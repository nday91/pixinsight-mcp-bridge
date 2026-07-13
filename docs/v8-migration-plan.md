# Migration Plan: PJSR SpiderMonkey → V8 (PixInsight 1.9.4+)

Companion to [`v8-porting-guide.md`](./v8-porting-guide.md) (the general reference). This file is specific to this repo: what actually needs to change, where, and in what order.

## Scope

Only the PJSR (in-PixInsight) code targets the SpiderMonkey/V8 engine. The Node.js bridge server is out of scope — it already runs on Node's own V8, unrelated to PixInsight's embedded runtime, and requires no changes for this migration.

**In scope:**
- `src/pixinsight-mcp-bridge.js` — main PJSR entry point, ES5, uses `__base__`/prototype inheritance for the dialog
- `src/lib/handlers.jsh` — PJSR include file, ES5, command dispatcher (also uses prototype-based "classes")
- `test/test-handlers.js` — mocks the PJSR API; mocks must still match real API shape after migration (e.g. constant namespacing)

**Out of scope (no change needed):**
- `src/bridge/server.js`, `src/bridge/mcp-handler.js`, `src/bridge/ipc.js` — pure Node.js, run outside PixInsight
- `test/run-tests.js`, `test/test-server.js`, `test/test-mcp-handler.js` — Node.js test files

## Current SpiderMonkey-specific constructs found

From a grep of `src/`:

1. **`__base__` prototype inheritance** (`pixinsight-mcp-bridge.js:361-362, 528`)
   - `MCPBridgeDialog` is built as `function MCPBridgeDialog() { this.__base__ = Dialog; this.__base__(); ... }` with `MCPBridgeDialog.prototype = new Dialog();` at file scope.
   - This pattern is non-functional under V8 and must become an ES6 `class MCPBridgeDialog extends Dialog`.

2. **Legacy underscore constants** (`pixinsight-mcp-bridge.js:380,418,426,434`)
   - `TextAlign_Right`, `TextAlign_VertCenter` → `TextAlignment.Right`, `TextAlignment.VertCenter`.

3. **`DataType_String`** (`pixinsight-mcp-bridge.js:90,102`)
   - Used in `Settings.read(...)` / `Settings.write(...)`. Legacy underscore form; needs verifying against the live 1.9.4 API for its namespaced replacement (likely `DataType.String`) since it wasn't in the guide's explicit mapping table.

4. **`CommandDispatcher` in `handlers.jsh`** uses `function CommandDispatcher() {...}` + `CommandDispatcher.prototype.xxx = function() {...}` throughout. This is plain ES5 prototype-method style (no `__base__`, no inheritance from a PixInsight-native class) — it is valid, unaffected JS and does **not** strictly need to become a `class`, but converting it improves consistency and readability. Lower priority than #1.

5. **No `#include <pjsr/...>` directives found** — `handlers.jsh` is pulled in via a project-local `#include "lib/handlers.jsh"`, which is fine and unrelated to the deprecated `<pjsr/...>` header includes. No changes needed there.

6. **No `gc()`, `VectorGraphics`, `ImageStatistics`, `Image.forEach*`, `Control.showAlias/hideAlias`, `Compression.*`, or `.prototype.CONSTANT` process-parameter usage found** in current source — these are guide-documented pitfalls that don't currently apply to this codebase, but keep an eye out if new code is added.

7. **Missing `#engine` directive and version guard** — the script currently has no `#engine` directive (defaults to legacy `sm`) and no `CoreApplication.ensureMinimumVersion(1, 9, 4)` call.

## Plan

### Phase 0 — Decide target compatibility — DECIDED
- [x] Hard cutover to V8-only, PixInsight ≥1.9.4 required. No SpiderMonkey compatibility maintained. ES6 classes used throughout.

### Phase 1 — Engine declaration — DONE
- [x] Added `#engine v8` as the first non-comment directive of `src/pixinsight-mcp-bridge.js`.
- [x] Added `CoreApplication.ensureMinimumVersion( 1, 9, 4 );` right after the `#include`.
- [x] Header comment and `Requirements` block updated to say PixInsight 1.9.4+ / V8.

### Phase 2 — Convert `MCPBridgeDialog` to an ES6 class — DONE
- [x] `MCPBridgeDialog` is now `class MCPBridgeDialog extends Dialog { constructor() { super(); ... } }`.
- [x] All former `MCPBridgeDialog.prototype.xxx = function(...) {...}` methods (`_addProcessToTree`, `_onAddProcess`, `_onRemoveProcess`, `_onStart`, `_onStop`, `_onClose`) are now class methods.
- [x] Removed the trailing `MCPBridgeDialog.prototype = new Dialog();` line.
- [x] Used a `class` declaration (not a `var X = class ...` expression) since the script runs under `#engine v8` (isolated runtime) — redeclaration risk only applies to `v8-private`/`v8-default`.
- [x] **Extra (beyond original plan):** also converted `IPCProcessor` and `MCPBridgeController` — both used the plain ES5 `function` + `.prototype.x = function` pattern (no `__base__`, not strictly broken under V8) — to ES6 `class` for consistency, since the whole file is being modernized anyway.

### Phase 3 — Fix legacy constants — DONE
- [x] `TextAlign_Right | TextAlign_VertCenter` (4 occurrences) → `TextAlignment.Right | TextAlignment.VertCenter`.
- [x] `DataType_String` (2 occurrences) → `DataType.String`. Not independently verified against a live PixInsight 1.9.4 install/API docs (no access in this session) — confirm during Phase 6 manual testing; this was the one guessed mapping not explicitly listed in the porting guide.
- [x] Re-grepped `src/` for any other legacy `Foo_Bar`-shaped constants — none remaining.

### Phase 4 — `handlers.jsh` cleanup — DONE
- [x] `CommandDispatcher` converted from `function` + `.prototype.x = function` to `class CommandDispatcher { ... }`.
- [x] `eval(entry.id)` / `eval(processId)` left as-is — dynamic constructor lookup by name is unaffected by the engine migration.

### Phase 5 — Update mocks and tests — DONE
- [x] `test/test-handlers.js` didn't stub any `TextAlign_*`/`DataType_*` constants (those only live in `pixinsight-mcp-bridge.js`, which the Node test harness doesn't load), so no mock renames were needed.
- [x] Fixed a real harness bug surfaced by the `CommandDispatcher` → `class` conversion: Node's `vm` module does not attach top-level `class` declarations as own properties of the context's global object (unlike `var`/`function`), so `sandbox.CommandDispatcher` was `undefined` after `vm.runInContext`. Fixed by pulling the binding out explicitly: `sandbox.CommandDispatcher = vm.runInContext("CommandDispatcher", sandbox);` in `loadHandlers()`. All 80 tests pass (`npm test`).
- [x] Also did a standalone V8 syntax check of both `pixinsight-mcp-bridge.js` (with PJSR-only preprocessor lines stripped and PJSR globals stubbed) and `handlers.jsh` via `vm.Script` — both parse cleanly.
- [ ] These are still Node-side tests that mock the PJSR API — they don't run through PixInsight's actual embedded runtime, so they won't catch real in-app errors (e.g. whether `extends Dialog` truly behaves as documented against PixInsight's native `Dialog` class, or whether `DataType.String` is the correct real constant name). Manual in-app testing (Phase 6) is still required.

### Phase 6 — Manual validation inside PixInsight 1.9.4+ — DONE (2026-07-13)
- [x] Loaded the script in a real PixInsight 1.9.4+ install via Script > Feature Scripts. Ran from the deployed copy at `C:/Program Files/PixInsight/src/scripts/MCP-Bridge/`.
- [x] `MCPBridgeDialog extends Dialog` constructs and renders correctly — script printed its startup banner and reached `findNodePath()` without any exception, which confirms the dialog/class construction path works at runtime.
- [x] `DataType.String` confirmed correct — no error/exception from `Settings.read`/`Settings.write` on script load (would have thrown immediately if wrong). Phase 3's guessed mapping was correct.
- [x] Found and fixed one unrelated pre-existing deprecation surfaced by this run: `src/pixinsight-mcp-bridge.js` used the deprecated global `corePlatform` in `findNodePath()` (line 63) instead of `System.platform`, triggering `** Warning: corePlatform is deprecated: Use System.platform instead.` in the Process Console. Fixed to `var platform = System.platform;` while keeping the same `"MSWINDOWS"`/`"Windows"` string check for robustness (exact `System.platform` return values aren't confirmed against docs, so the check stays defensive). Not part of the SpiderMonkey→V8 engine migration itself, but fixed while touching this file. Redeployed to the PixInsight scripts folder 2026-07-13.
- [x] "Node.js not found!" confirmed to be a stale-PATH issue, not a bug — after restarting PixInsight, Node.js v24.18.0 was detected successfully.
- [x] Dialog controls (`Label`, `SpinBox`, `TreeBox`, `Edit`, `PushButton`, `GroupBox`) verified working interactively.
- [x] Full flow exercised successfully: Start bridge → Node.js discovery, server spawn, IPC polling → Add/Remove custom process → persisted across restart → Stop bridge → Close dialog.
- [x] Watched the Process Console for deprecation warnings — found and fixed the `corePlatform` one above; no other deprecation warnings seen.

### Phase 7 — Docs — DONE
- [x] `README.md` updated: PixInsight requirement bumped to "1.9.4 or later (V8 JavaScript runtime)"; architecture section and project-structure tree updated from "ES5" to "V8"; "ES5 compliance" bullet under Key Design Decisions replaced with a "V8 runtime" bullet describing the `#engine v8` directive and modern ECMAScript usage.

## Status

All phases (0–7) are complete. The SpiderMonkey→V8 migration has been validated end-to-end against a real PixInsight 1.9.4+ install: dialog construction, `DataType.String`, Node.js discovery, and the full start/stop/add/remove/persist bridge flow all work correctly. Changes are still uncommitted to git on branch `V8conv` — ask before committing.
