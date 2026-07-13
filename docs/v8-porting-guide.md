# PixInsight V8 JavaScript Runtime — Script Porting Guide (reference)

Source: https://pixinsight.net/dev/index.php?ams/the-new-v8-javascript-runtime-in-pixinsight-1-9-4-script-porting-guide.13/&full=1

Saved 2026-07-13 for reference across sessions. This is a saved copy of the article content (fetched via WebFetch), not a live mirror — re-fetch the URL if something here seems stale or incomplete.

## Overview

PixInsight 1.9.4 replaces the legacy SpiderMonkey 24 engine (from 2014) with Google's V8 JavaScript runtime. The migration improves performance dramatically while modernizing the language to ECMAScript 2025 standards.

## Runtime Selection

Scripts must declare their target engine using the `#engine` preprocessor directive at the file's beginning:

**Engine Options:**
- `#engine v8` or `#engine v8-new`: Creates isolated runtime destroyed after script termination (~20-30ms initialization)
- `#engine v8-default`: Uses persistent root engine (ideal for testing, risky for production)
- `#engine v8-private`: Creates cached private runtime reused across executions
- `#engine sm`: Legacy SpiderMonkey (default if no directive specified)

Example:
```javascript
#engine v8
#feature-id Ephemerides : Ephemerides > Ephemerides
CoreApplication.ensureMinimumVersion( 1, 9, 4 );
```

## Critical Migration Steps

### 1. Remove Legacy Include Directives

All `#include <pjsr/...>` header files are deprecated. Constants previously defined in `.jsh` files are now available as static class properties.

**Before:**
```javascript
#include <pjsr/StdIcon.jsh>
let icon = StdIcon_NoIcon;
```

**After:**
```javascript
let icon = StdIcon.NoIcon;
```

**Common Constant Mappings:**
- `Cipher_*` → `CipherAlgorithm.*`
- `Compression_*` → `CompressionAlgorithm.*`
- `FileType_*`, `File_Attribute_*`, `FilePermission_*` → `FileFlag.*`
- `GradientSpread_*` → `GradientSpreadMode.*`
- `Interpolation_*` → `InterpolationAlgorithm.*`
- `Key_*` → `KeyCode.*`
- `MorphOp_*` → `MorphologicalOp.*`
- `RBFType_*` → `RadialBasisFunction.*`
- `ReadTextOptions_*` → `ReadTextOption.*`
- `SampleType_*` → `PixelSampleType.*`
- `TextAlign_*` → `TextAlignment.*`

(Note: `DataType_*` constants, e.g. `DataType_String` used by `Settings.read`/`Settings.write`, follow the same pattern — verify the new namespaced form, likely `DataType.String`, against the live API docs when doing the conversion.)

### 2. Convert Constructor Functions to ES6 Classes

V8 doesn't support SpiderMonkey's prototype-based inheritance pattern using `__base__` properties.

**SpiderMonkey Pattern (Non-functional in V8):**
```javascript
function FooDialog( foo, bar ) {
   this.__base__ = Dialog;
   this.__base__();
   this.foo = foo;
   this.bar = (bar !== undefined) ? bar : 42;
   this.kung = function( foo ) { ... };
}
FooDialog.prototype = new Dialog;
```

**V8 Class Pattern (Required):**
```javascript
class FooDialog extends Dialog {
   constructor( foo, bar = 42 ) {
      super();
      this.foo = foo;
      this.bar = bar;
   }
   kung( foo ) { ... }
}
```

**Important:** For scripts using private runtimes (`v8-private` or `v8-default`), prefer class expressions over declarations to avoid redeclaration errors:

```javascript
var FooDialog = class extends Dialog {
   constructor( foo, bar = 42 ) {
      super();
      this.foo = foo;
      this.bar = bar;
   }
   kung( foo ) { ... }
};
```

### 3. Remove Process Prototype Property Access

Constants defined by processes must access class constructors directly, not prototypes.

**Before (Invalid in V8):**
```javascript
SA.mode = StarAlignment.prototype.OutputMatrix;
SA.rbfType = StarAlignment.prototype.DDMThinPlateSpline;
```

**After (Correct):**
```javascript
SA.mode = StarAlignment.OutputMatrix;
SA.rbfType = StarAlignment.DDMThinPlateSpline;
```

## Breaking Changes & Deprecated Functions

### Removed: gc() Method

V8's autonomous garbage collector cannot be forced. The `gc()` call now produces deprecation warnings with no effect.

**Memory Management Strategy:**

Instead of relying on garbage collection, explicitly deallocate large objects:

**Before:**
```javascript
function zoomedRendition( bitmap, zoomFactor ) {
   return bitmap.toImage().render( zoomFactor, false, true );
}
let zoomedBitmap = zoomedRendition( sourceBitmap, zoomFactor );
gc();
```

**After:**
```javascript
function zoomedRendition( bitmap, zoomFactor ) {
   let image = bitmap.toImage();
   let zoomedBitmap = image.render( zoomFactor, false, true );
   image.free();  // Explicit deallocation
   return zoomedBitmap;
}
let zoomedBitmap = zoomedRendition( sourceBitmap, zoomFactor );
zoomedBitmap.clear();  // Deallocate when done
```

### Deprecated: Most Global.* Properties

Only these remain non-deprecated:
- `void cerr( String text )`
- `void cerrln( String text )`
- `void cflush()`
- `void cout( String text )`
- `void coutln( String text )`
- `String format( String fmt, ... )`

Other global extensions moved to: `CoreApplication`, `Runtime`, `File`, `System` classes.

### Removed Classes & Methods

**VectorGraphics:** Replace with `Graphics` (now handles all coordinate types automatically)

**ImageStatistics:** Functionality moved to Image class methods:
- `Image.median()`
- `Image.MAD()`
- `Image.stdDev()`
- `Image.rangeClippingEnabled`

**Image iteration methods removed:**
- `Image.forEachSample()`
- `Image.forEachMutableSample()`
- `Image.forEachPixel()`
- `Image.forEachMutablePixel()`

Use `ImageIterator` class instead for native-speed pixel access.

**Control methods removed:**
- `Control.showAlias()`
- `Control.hideAlias()`

### Compression API Changes

**Before:**
```javascript
let result = Compression.compress( data );  // Returns array of arrays
let decompressed = Compression.uncompress( result );
```

**After:**
```javascript
let subblocks = Compression.compress( data );  // Returns array of objects
// Each subblock: { compressedData, uncompressedSize, checksum }
let decompressed = Compression.uncompress( subblocks );
```

### Array Return Type Changes

**EphemerisFile.objects** and **EphemerisFile.visibleObjects()** now return arrays of objects instead of arrays of arrays.

**FileFormatInstance.open():** Returns empty array `[]` when file has no readable images (previously returned null); returns `null` only for I/O failures.

### View/ImageWindow Property Changes

- `ImageWindow.previewById()`: Returns `null` (not invalid View object) if preview not found
- `View.properties`: Returns empty array (not null) for invalid views
- `View.viewById()`: Returns `null` for missing views
- `View.propertyAttributes()`: Returns `PropertyAttribute.Invalid` (not null) when property missing
- `View.propertyType()`: Returns `PropertyType.Invalid` (not null) when property missing
- `View.window`: Returns `null` (not invalid ImageWindow) for invalid views
- `View.image`: Returns `null` (not invalid Image) for invalid views

### ImageWindow.purge() Signature Change

**Before:**
```javascript
ImageWindow.purge( true, true, true );  // Accepted histogram & property args
```

**After:**
```javascript
ImageWindow.purge();  // No arguments; properties managed automatically
```

### FileFormat Methods Removed

Eliminated: `formatSpecificData`, `usesFormatSpecificData()`, `validateFormatSpecificData()`, `disposeFormatSpecificData()`. Format-specific data now managed automatically.

## New PJSR Classes

### FMath: High-Performance Mathematics

Accelerated math via WebAssembly achieving order-of-magnitude speedup over standard Math:

```javascript
let value = FMath.mtf( balance, input );  // Midtones transfer function
let result = FMath.sqrt( x );
// All standard Math functions available, plus specialized methods
```

### Stat: Statistical Calculations

Replaces deprecated Math object extensions; provides organized statistical methods:

```javascript
let stdDev = Stat.stdDev( data );
let median = Stat.median( data );
```

### ImageIterator: Native-Speed Pixel Access

Direct access to pixel buffers via typed arrays without JavaScript bridge overhead:

```javascript
let I = new ImageIterator( image, channelIndex );
for ( let y = 0; y < I.height; ++y )
   for ( let x = 0; x < I.width; ++x )
      I[y][x] = FMath.mtf( balance, I[y][x] );
```

**For integer images**, use conversion methods:

```javascript
function processInteger( I, m ) {
   for ( let c = 0; c < I.length; ++c )
      for ( let y = 0; y < I[c].height; ++y )
         for ( let x = 0; x < I[c].width; ++x )
            I[c][y][x] = I[c].toSample(
               FMath.mtf( m[c], I[c].toReal( I[c][y][x] ) )
            );
}
```

### StarDetector: Star Detection Algorithms

C++ core implementation replaces JavaScript version; no longer requires include:

```javascript
let D = new StarDetector;
D.fitPSF = true;  // Enable elliptical Gaussian fitting
let stars = D.stars( image );  // Returns array of StarData objects
```

### PSF: Point Spread Function Fitting

Levenberg-Marquardt fitting with multithreading:

```javascript
let D = new StarDetector;
let stars = D.stars( image );
let P = PSF.fitStars( image, stars, PSFunction.Auto );
// Each element: { B, A, x, y, fwhmX, fwhmY, theta, function, beta, signal, mad }
```

### BRQuadTree: Spatial Partitioning

Efficient 2D geometric search structure; no longer requires include:

```javascript
let tree = new BRQuadTree( ... );  // Available directly as core class
```

### Matrix, Vector, Point, Rect: Reimplemented in JavaScript

Pure JavaScript implementations with high performance; support natural syntax:

```javascript
let ref_F_I = new Matrix(
   1,  0,              -0.5,
   0, -1, this.height + 0.5,
   0,  0,               1 );

let ref_F_G = this.ref_I_G_linear.mul( ref_F_I );
wcs.cd1_1 = ref_F_G[0][0];  // Adjacent subscripts for matrix access
wcs.cd1_2 = ref_F_G[0][1];

let orgF = ref_F_G.inverse().apply( new Point( 0, 0 ) );
```

### XML Support Classes

Complete XML processing functionality matching PCL/C++:

```javascript
let xml = new XMLDocument;
xml.xml = new XMLDeclaration( "1.0", "UTF-8" );
xml.rootElement = new XMLElement( "root" )
   .setAttribute( "version", "1.0" )
   .addChildNode( new XMLElement( "child" )
      .addChildNode( new XMLText( "content" ) ) );
xml.autoFormatting = true;
xml.serializeToFile( "/tmp/output.xml" );
```

### System: Host Machine Properties

Access OS and hardware information:

```javascript
JSON.stringify( System.physicalMemoryStatus() );
// Output: {"totalBytes":810916085760,"availableBytes":785220763648}
```

## Limitations

### Unsupported: JavaScript Modules

ECMAScript 6 modules (import/export syntax) are not supported. Use the existing C-style preprocessor with `#include` directives instead:

```javascript
#include "MyLibrary.js"
#include <pjsr/Math.jsh>
```

### Preprocessor/Private Class Field Conflicts

ECMAScript 2022 private class fields use `#` syntax (e.g., `#myProperty`). The PixInsight preprocessor also uses `#` for directives. Avoid naming private fields using reserved preprocessor identifiers:

```
#define, #else, #endif, #engine, #error, #feature-icon, #feature-id,
#feature-info, #if, #ifdef, #ifeq, #ifgt, #ifgteq, #iflt, #iflteq,
#ifndef, #ifneq, #ifnoneof, #ifoneof, #include, #script-id, #undef, #warning
```

Safe example:
```javascript
class Bar {
   #bar = 42;  // OK: 'bar' not a reserved identifier
   #define = 5;  // CONFLICT: avoid this
}
```

## Future Roadmap

Planned enhancements:
- Multithreaded execution with worker threads
- Non-modal dialog interfaces integrated with PixInsight
- Enhanced image processing classes
- Mixed JavaScript/C++ execution model
- Integrated JavaScript debugger

## Summary of Key Migration Checklist

1. Add `#engine v8` directive at script start
2. Add `CoreApplication.ensureMinimumVersion( 1, 9, 4 );` call
3. Remove all `#include <pjsr/...>` directives
4. Convert constructor functions to ES6 classes with `extends`
5. Replace `ClassName.prototype.CONSTANT` with `ClassName.CONSTANT`
6. Remove `gc()` calls; use explicit `object.free()` or `object.clear()`
7. Replace VectorGraphics with Graphics
8. Move Image statistics to Image class methods
9. Replace Image iteration methods with ImageIterator
10. Update Compression API to handle subblock objects
11. Test View/ImageWindow property access patterns
12. Update Memory management strategies for V8's non-deterministic garbage collection
