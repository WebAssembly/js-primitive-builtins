# JS Primitive Builtins Proposal

## Summary

Building on top of the JS String Builtins proposal, this proposal extends the set of available builtins to manipulate other JS primitive data types: `number`, `boolean`, `symbol`, `bigint` and `undefined`.

## Motivation

The [JS String Builtins proposal](https://github.com/WebAssembly/js-string-builtins) introduced an efficient mechanism to manipulate JS strings from WebAssembly code using the JS embedding.
It focused on strings as a high-impact first step.
Now that infrastructure for JS builtins is firmly in place, this proposal suggests an expanded set of builtins that manipulate JS primitive values.

The motivation is largely the same as for the original JS string builtins.
All of the proposed builtins can be *expressed* today using JS glue code, but we want to avoid the cost of a Wasm-to-JS function call for operations that should be a tight sequence of inline instructions.

## Overview

Most of what we could say here has already been very well laid out [in the overview of the JS String Builtins proposal](https://github.com/WebAssembly/js-string-builtins/blob/main/proposals/js-string-builtins/Overview.md).
I do not think it is worth repeating them here.

In particular, the *goals* for builtins remain the same.
Builtins should be simple and only provide functionality that is semantically already provided by JavaScript.

The proposed list of builtins is based on experience with the Scala.js-to-Wasm compiler.
Through benchmarking and profiling, we have identified a number of operations that show up high on profiles for no good reason, other than they require glue code to JavaScript.
We have extrapolated to some additional operations that we think are likely relevant to other toolchains.

Here is a quick overview of the builtins we propose.
Currently, the set is intentionally fairly broad, so that we can discuss what is actually useful and what might be overreaching.
In the expanded specifications below, we will mark the functions that we consider particularly important, based on our experience with Scala.js.

* String (extensions to the existing `wasm:js-string`):
    * Conversion from primitive numeric types: `fromI32`, `fromU32`, `fromI64`, `fromU64`, `fromF64`
    * Unicode-aware case conversions: `toLowerCase`, `toUpperCase`
* Number (`wasm:js-number`):
    * Type test: `test`, `testI32`, `testU32`
    * Create from primitive: `fromF64`, `fromI32`, `fromU32`
    * Extract to primitive: `toF64`, `toI32`, `toU32`
* Boolean (`wasm:js-boolean`):
    * Type test: `test`
    * (creation is efficiently achieved by importing `true` and `false` as `global`s)
    * Extract to primitive: `toI32`
* Undefined (`wasm:js-undefined`):
    * Type test: `test`
    * (creation is efficiently achieved by importing `void 0` as a `global`)
* Symbol (`wasm:js-symbol`):
    * Type test: `test`
    * Identity test: `equals`
    * Creation as if by `Symbol(description)` or `Symbol.for(key)`
    * Extraction of the `description` and the `key`
* Bigint (`wasm:js-bigint`):
    * Type test: `test`
    * Create from primitive: `fromF64`, `fromI64`, `fromU64`
    * Extract to primitive: `convertToF64`, `wrapToI64`
    * Operations: `add`, `pow`, `shl`, etc. (the ones corresponding to JS operators)
    * Wrapping operations: `asIntN`, `asUintN`
    * Parsing: `parse`
    * Conversion to string: `toString`

## About the "universal representation"

Many of the rationales below talk about a "universal representation".
This is in the context of a language that compiles to Wasm in a JS embedding, and wants to interoperate with JavaScript at a fairly deep level.
In such a language, a universal representation of values must be compatible with JavaScript.
For example, primitive numbers stored in a universal representation must appear as JS `number`s when seen by JavaScript code.

In more details, the "boxing" operation from `i32` to `anyref` (or `externref`) followed by `ToJSValue` must commute (as category theorists say) with a direct conversion of `i32` through `ToJSValue`.
The following diagram illustrates the issue:

```
         upcast/
 ┌─────┐   downcast ┌────────┐
 │ i32 ◄────────────► anyref │
 └───▲─┘            └───▲────┘
     │                  │
     │                  │ToJSValue/
     │                  │  ToWebAssemblyValue
     │              ┌───▼────┐
     └──────────────►  JS    │
ToJSValue/          └────────┘
  ToWebAssemblyValue
```

The toolchain has direct control over the `upcast/downcast` arrows.
It may also insert custom code around the direct `i32`<->JS conversion.
However, the conversions between `anyref` and JS are dictated by the JS embedding and cannot be amended.
That means the universal representation of an `i32` must already be a JS `number`.

For more context, you may want to revisit [the notes](https://github.com/WebAssembly/meetings/blob/main/main/2024/CG-06.md#experience-report-compiling-scalajs-to-wasm-s%C3%A9bastien-doeraene) and [slides](https://docs.google.com/presentation/d/1PPAyjOAJrSi6HvZ39uknIDX0fn23u9I8MWLWw9RQMpE/edit?usp=sharing) of the Scala.js experience report that we gave during the June 2024 hybrid CG meeting.

## Open questions

### `toString` with radix for `i32` and `i64`

Should we expose the integer ormatting methods that take an explicit radix?

For `i32`, this corresponds to `Number.prototype.toString(radix)`.
V8 already contains dedicated code to recognize it, when hidden behind a bound `Function.prototype.call`; that suggests that there is already a strong incentive to support conversion of `i32` to string with radix as a builtin.

For `i64`, that is not technically supported by JS today.
JS does not have radix support for `bigint`s, and the way `i64` features are justified in this proposal is that they are presented as `bigint`s on the JS side.

The inconcistency between `i32` and `i64` suggests *not* to include them at this time.

Regardless, it is not a goal to provide full `bigint` support here.

## Specifications

In this section, we give the specification of the proposed builtins.
We follow the same style as [the JS String builtins proposal](https://github.com/WebAssembly/js-string-builtins/blob/main/proposals/js-string-builtins/Overview.md#js-string-builtin-api).
In particular, we also use the `trap()` function to specify that a Wasm trap should occur.

### "wasm:js-string" "fromI32"

```js
func fromI32(
  x: i32
) -> (ref extern) {
  return "" + x;
}
```

### "wasm:js-string" "fromU32"

```js
func fromU32(
  x: i32
) -> (ref extern) {
  // NOTE: `x` is interpreted as signed 32-bit integers when converted to a JS
  // value using standard conversions. Reinterpret it as unsigned here.
  x >>>= 0;

  return "" + x;
}
```

### "wasm:js-string" "fromI64"

```js
func fromI64(
  x: i64
) -> (ref extern) {
  // NOTE: `x` is interpreted as a signed JS `bigint`, which provides the
  // expected conversion to string.
  return "" + x;
}
```

### "wasm:js-string" "fromU64"

```js
func fromU64(
  x: i64
) -> (ref extern) {
  // NOTE: `x` is interpreted as a signed JS `bigint`. Reinterpret it as
  // unsigned here.
  x = BigInt.asUintN(64, x);

  return "" + x;
}
```

### "wasm:js-string" "fromF64"

```js
func fromF64(
  x: f64
) -> (ref extern) {
  return "" + x;
}
```

### "wasm:js-string" "toLowerCase"

```js
func toLowerCase(
  string: externref
) -> (ref extern) {
  if (typeof string !== "string")
    trap();

  return string.toLowerCase();
}
```

### "wasm:js-string" "toUpperCase"

```js
func toUpperCase(
  string: externref
) -> (ref extern) {
  if (typeof string !== "string")
    trap();

  return string.toUpperCase();
}
```

We include specifically `toLowerCase()` and `toUpperCase()`, but not the plethora of other methods of strings.
Other methods can already be efficiently implemented using the existing builtins in `js-string`, on the Wasm side.
The Unicode-aware case conversions (using the ROOT locale) require to embed a significant subset of the Unicode database.
That's not something we want to ship along with our Wasm payloads.

### "wasm:js-number" "test"

```js
func test(
  x: externref
) -> i32 {
  if (typeof x !== "number")
    return 0;
  return 1;
}
```

### "wasm:js-number" "testI32"

Technically implementable using `testF64`, `toF64`, and additional Wasm instructions.
However, we suspect there are many cases where engines would first have to convert an integer stored internally into a float before giving it to Wasm, so that would incur two useless conversions.

```js
func testI32(
  x: externref
) -> i32 {
  if (typeof x !== "number")
    return 0;
  if ((x | 0) !== x || Object.is(x, -0))
    return 0;
  return 1;
}
```

When the result is expected to be `1`, in today's engines, it is worth first trying `(ref.test (ref i31) x)` before falling back on JS glue code.

### "wasm:js-number" "testU32"

```js
func testU32(
  x: externref
) -> i32 {
  if (typeof x !== "number")
    return 0;
  if ((x >>> 0) !== x || Object.is(x, -0))
    return 0;
  return 1;
}
```

### "wasm:js-number" "fromF64"

```js
func fromF64(
  x: f64
) -> (ref extern) {
  return x;
}
```

### "wasm:js-number" "fromI32"

```js
func fromI32(
  x: i32
) -> (ref extern) {
  return x;
}
```

In today's engines, it is worth spending Wasm instructions to test whether `x` fits in 31 bits.
If it does, use `(ref.i31 x)`.
Otherwise, fall back on JS glue code.

### "wasm:js-number" "fromU32"

```js
func fromU32(
  x: i32
) -> (ref extern) {
  // NOTE: `x` is interpreted as signed 32-bit integers when converted to a JS
  // value using standard conversions. Reinterpret it as unsigned here.
  x >>>= 0;

  return x;
}
```

### "wasm:js-number" "toF64"

```js
func toF64(
  x: externref
) -> f64 {
  if (typeof x !== "number")
    trap();

  return x;
}
```

### "wasm:js-number" "toI32"

Technically implementable using `toF64`, and additional Wasm instructions.
However, we suspect there are many cases where engines would first have to convert an integer stored internally into a float before giving it to Wasm, so that would incur two useless conversions.

```js
func toI32(
  x: externref
) -> i32 {
  if (typeof x !== "number")
    trap();

  if ((x | 0) !== x || Object.is(x, -0))
    trap();

  return x;
}
```

In today's engines, it is worth first trying a `br_on_cast (ref i31)`, with a fallback on JS glue code.

### "wasm:js-number" "toU32"

```js
func toU32(
  x: externref
) -> i32 {
  if (typeof x !== "number")
    trap();

  if ((x >>> 0) !== x || Object.is(x, -0))
    trap();

  return x;
}
```

### "wasm:js-boolean" "test"

```js
func test(
  x: externref
) -> i32 {
  if (typeof x !== "boolean")
    return 0;
  return 1;
}
```

### "wasm:js-boolean" "toI32"

```js
func toI32(
  x: externref
) -> i32 {
  if (typeof x !== "boolean")
    trap();

  if (x)
    return 1;
  return 0;
}
```

### "wasm:js-undefined" "test"

```js
func test(
  x: externref
) -> i32 {
  if (typeof x !== "undefined")
    return 0;
  return 1;
}
```

### "wasm:js-symbol" "test"

```js
func test(
  x: externref
) -> i32 {
  if (typeof x !== "symbol")
    return 0;
  return 1;
}
```

### "wasm:js-symbol" "equals"

```js
func equals(
  x: externref,
  y: externref
) -> i32 {
  if (typeof x !== "symbol" && x !== null)
    trap();
  if (typeof y !== "symbol" && y !== null)
    trap();
  return x === y;
}
```

### "wasm:js-symbol" "unique"

```js
func unique(
  description: externref
) -> externref {
  if (typeof description !== "string" && description !== null)
    trap();
  if (description === null)
    return Symbol();
  return Symbol(description);
}
```

### "wasm:js-symbol" "for"

```js
func forKey(
  key: externref
) -> externref {
  if (typeof key !== "string")
    trap();
  return Symbol.for(key);
}
```

### "wasm:js-symbol" "description"

```js
func description(
  x: externref
) -> externref {
  if (typeof x !== "symbol")
    return trap();
  const description = x.description;
  if (typeof description === 'undefined')
    return null;
  return description;
}
```

### "wasm:js-symbol" "keyFor"

```js
func keyFor(
  x: externref
) -> externref {
  if (typeof x !== "symbol")
    return trap();
  const key = Symbol.keyFor(x);
  if (typeof key === 'undefined')
    return null;
  return key;
}
```

### "wasm:js-bigint" "test"

```js
func test(
  x: externref
) -> i32 {
  if (typeof x !== "bigint")
    return 0;
  return 1;
}
```

### "wasm:js-bigint" "fromF64"

```js
func fromF64(
  x: f64
) -> (ref extern) {
  // NOTE: BigInt(x) would throw a RangeError in the situation below.
  // Trap instead.
  if (!Number.isInteger(x))
    trap();

  return BigInt(x);
}
```

### "wasm:js-bigint" "fromI64"

```js
func fromI64(
  x: i64
) -> (ref extern) {
  // NOTE: `x` is interpreted as a signed JS `bigint` by the Wasm-to-JS
  // interface, so this appears as a no-op.
  return x;
}
```

### "wasm:js-bigint" "fromU64"

```js
func fromU64(
  x: i64
) -> (ref extern) {
  // NOTE: `x` is interpreted as a signed JS `bigint` by the Wasm-to-JS
  // interface. Reinterpret it as unsigned.
  return BigInt.asUint(64, x);
}
```

### "wasm:js-bigint" "convertToF64"

```js
func convertToF64(
  x: externref
) -> f64 {
  if (typeof x !== "bigint")
    trap();

  return Number(x);
}
```

### "wasm:js-bigint" "wrapToI64"

```js
func wrapToI64(
  x: externref
) -> i64 {
  if (typeof x !== "bigint")
    trap();

  // NOTE: ToWebAssemblyValue specifies a wrapping conversion via ToBigInt64
  return x;
}
```

### "wasm:js-bigint" "add" (and other JS operators)

```js
func add(
  x: externref,
  y: externref
) -> f64 {
  if (typeof x !== "bigint")
    trap();
  if (typeof y !== "bigint")
    trap();

  return x + y;
}
```

### "wasm:js-bigint" "asIntN"

```js
func asIntN(
  bits: i32,
  bigint: externref
) -> f64 {
  if (typeof bigint !== "bigint")
    trap();

  // NOTE: `bits` is interpreted as signed 32-bit integers when converted to a
  // JS value using standard conversions. Reinterpret it as unsigned here.
  bits >>>= 0;

  return BigInt.asIntN(bits, bigint);
}
```

### "wasm:js-bigint" "asUintN"

```js
func asUintN(
  bits: i32,
  bigint: externref
) -> f64 {
  if (typeof bigint !== "bigint")
    trap();

  // NOTE: `bits` is interpreted as signed 32-bit integers when converted to a
  // JS value using standard conversions. Reinterpret it as unsigned here.
  bits >>>= 0;

  return BigInt.asUintN(bits, bigint);
}
```

### "wasm:js-bigint" "parse"

```js
func parse(
  string: externref
) -> f64 {
  if (typeof string !== "string")
    trap();

  // NOTE: ToBitInt(argument) throws a SyntaxError if the string is not a valid
  // integer string. However, that is not easy to test ahead of time. So we
  // catch the SyntaxError instead to turn it into a Wasm trap.
  try {
    return BigInt(string);
  } catch (e) {
    // Assert: e instanceof SyntaxError
    trap();
  }
}
```

### "wasm:js-bigint" "toString"

```js
func toString(
  bigint: externref
) -> f64 {
  if (typeof bigint !== "bigint")
    trap();

  return "" + bigint;
}
```

## Not included

This section keeps track of a few things that were considered at some point but removed, although a case could still be made to bring them back.

### Functions that can be imported as is

We considered builtins for `parseFloat`, `parseInt` and `Object.is`.
However, they can already be imported with acceptable signatures today, with their exact identity.
This means engines can already detect them, and optimize them if they deem it useful.

Likewise, we had considered methods of `Math` like `Math.sin`.

Caveat for `parseFloat` and `parseInt`: they may have to invoke arbitrary user-defined JS code through `ToString()` of their string argument.
Their builtin equivalent could have avoided that by only accepting real `string`s instead.
However, that was not deemed a good enough benefit.

### JS operators `x % y` and `x | 0`

These operators may receive (partial) support from dedicated hardward instructions: `fprem` for `x % y`, and the ARM-specific `FJCVTZS` for `x | 0`.
They could therefore be useful as builtins.
However, if we wanted to expose them to Wasm, actual Wasm opcodes would probably be a better avenue.
