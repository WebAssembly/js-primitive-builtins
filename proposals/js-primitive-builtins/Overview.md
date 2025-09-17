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

Some of those functions can be *directly* imported, because they do not rely on their `this` value (ex.: `parseFloat`, `sin`, `Object.is`).
These may already be optimized accurately by JS engines if they are used as is.
At least, their semantics are such that the engines are allowed to.
The purpose of including them here is to have some sort of guarantee that they *will* be optimized without a round-trip to JS land.

* String (extensions to the existing `wasm:js-string`):
    * Conversion from primitive numeric types: `fromI32`, `fromU32`, `fromI64`, `fromU64`, `fromF32`, `fromF64`
    * Unicode-aware case conversions: `toLowerCase`, `toUpperCase`
* Number (`wasm:js-number`):
    * Type test: `test`, `testF32`, `testI32`, `testU32`
    * Create from primitive: `fromF64`, `fromF32`, `fromI32`, `fromU32`
    * Extract to primitive: `toF64`, `toF32`, `toI32`, `toU32`
    * JS primitive operations: `fmod` (the `%` operator), `wrapToI32` (`x | 0`)
    * Math operations: `sin`, `cos`, etc. (the ones that have hardware equivalents, at least)
    * Parsing: `parse`
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
* Generic (`wasm:js-object`):
    * JS same-value: `is` (`Object.is`)
    * Conversion to string as if in string concat: `toString`

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

### Are the importable functions worth it?

`parseFloat`, `parseInt`, `Object.is`, `Math.sin` et al., can already be imported with acceptable signatures today.
Is it worth adding them as builtins?
Should we instead "strongly encourage" JS embeddings to recognize them at instantiation time and optimize them accordingly?

### Conversions of `bigint`s to/from bit arrays

Some languages with big integers provide direct conversions to/from bit arrays:

* as `i8` arrays, `i32` arrays and/or `i64` arrays,
* in little endian and/or big endian, and
* in two's complement representation or in sign-magnitude representation.

Should we also add all those conversions to `wasm:js-bigint`?

### `toString` and parsing with radix for `i32` and `i64`

Should we expose the integer parsing and formatting methods that take an explicit radix?

For `i32`, this corresponds to `parseInt(string, radix)` and `Number.prototype.toString(radix)`.
V8 already contains dedicated code to recognize the latter, when hidden behind a bound `Function.prototype.call`; that suggests that there is already a strong incentive to support conversion of `i32` to string with radix as a builtin.
`parseInt` is importable as is, as mentioned above.

For `i64`, that is not technically supported by JS today.
JS does not have radix support for `bigint`s, and the way `i64` features are justified in this proposal is that they are presented as `bigint`s on the JS side.
However, we may want to support them as builtins for consistency with `i32`.

It is not a goal to provide full `bigint` support here.

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

### "wasm:js-string" "fromF32"

Note: This can be implemented without additional overhead with `f64.promote_f32`+`fromF64`, so it is probably not needed.

```js
func fromF32(
  x: f32
) -> (ref extern) {
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

### "wasm:js-number" "testF32"

Could be implemented on the Wasm side using `testF64` and `toF64`.

```js
func testF32(
  x: externref
) -> i32 {
  if (typeof x !== "number")
    return 0;
  if (Math.fround(x) !== x && x === x) // note: do not reject NaN
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

### "wasm:js-number" "fromF32"

Note: Could be implemented with `f64.promote_f32` and `fromF64` without additional overhead, so it might not be needed, unless there is evidence that some engines convert 32-bit floats more efficiently than 64-bit floats.

```js
func fromF32(
  x: f32
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

### "wasm:js-number" "toF32"

Could be implemented with `toF64` and additional code on the Wasm side.

```js
func toF32(
  x: externref
) -> f32 {
  if (typeof x !== "number")
    trap();

  // NOTE: alternative semantics would be *not* to perform that test, and let
  // ToWebAssemblyValue demote to f32 instead. But then it is equivalent to
  // doing `toF64`+`f32.demote_f64`, so there is not much point.
  if (Math.fround(x) !== x && x === x)
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

### "wasm:js-number" "fmod"

Can be implemented in pure Wasm, although that misses the opportunity to leverage hardware `fprem` instructions, where they exist.

```js
func fmod(
  x: f64,
  y: f64
) -> f64 {
  return x % y;
}
```

### "wasm:js-number" "wrapToI32"

Can be implemented in pure Wasm, although that misses the opportunity to leverage hardware `FJCVTZS` instructions, where they exist.

```js
func wrapToI32(
  x: f64
) -> i32 {
  return x | 0;
}
```

Note that a hypothetical `wrapToU32` would be exactly equivalent, and hence is not proposed.

### "wasm:js-number" "sin" (and other similar `Math` operations)

Can be imported as is. Included for "guaranteed" no-glue-code calls.

Can also be implemented in pure Wasm, although that misses the opportunity to leveral relevant hardward support, where available.

```js
func sin(
  x: f64
) -> f64 {
  return Math.sin(x);
}
```

This set of builtins is particularly debated.
It has been suggested that compilers prefer software implementations for performance reasons.
If that is indeed the case, there is no real benefit to proposing them as builtins, as opposed to implementing them on the Wasm side.

### "wasm:js-number" "parse"

Can be imported as is as `parseFloat`, except that `parseFloat` calls `ToString()` on its argument, rather than rejecting non-strings.
The exact behavior of the proposed builtin can be achieved by calling `"wasm:js-string" "cast"` before calling `parseFloat`.
Included for "guaranteed" no-glue-code calls.

```js
func parse(
  string: externref
) -> f64 {
  if (typeof string !== "string")
    trap();

  return parseFloat(string);
}
```

Note: we do not propose `parseInt`, as it can be efficiently implemented on the Wasm side based on existing JS string builtins.
Moreover, languages tend to disagree on the specifics of the format anyway, which makes it a poor common ground.
`parseFloat` is more critical, as efficient implementations require big tables, which we do not want to ship along with our Wasm code.
At the same time, its semantics are more widely shared across languages.

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

### "wasm:js-object" "is"

This function is the closest thing to an identity test that applies to the universal representation on a JS host.

Can be imported as is.
Included for "guaranteed" no-glue-code calls.

```js
func is(
  x: externref,
  y: externref
) -> i32 {
  if (Object.is(x, y))
    return 1;
  return 0;
}
```

### "wasm:js-object" "toString"

```js
func toString(
  x: externref
) -> (ref extern) {
  return "" + x;
}
```

This builtin is probably the most controversial.
Unlike all the other builtins mentioned in this proposal, it *can* invoke arbitrary JS code.

It is also debatable whether to use string concatanation semantics (`"" + x`) or explicit conversion to string (`String(x)`).
This makes a difference for `symbol`s: they throw in the former case but succeed in the latter case.
This ambiguity adds to the debate of whether this builtin is worth it.
