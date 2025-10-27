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
We have extrapolated to operations that we think are likely relevant to other toolchains: conversions of *unsigned* integers (Scala happens to only have signed integer semantics in hot paths, but this is quite peculiar), and symbol equality (though that is still an open question).

Here is a quick overview of the builtins we propose.
The original set was fairly broad, so that we could discuss what is actually useful and what might be overreaching.
During Stage 1 discussions, the set was significantly reduced.

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
    * (creation is achieved by importing the functions `Symbol` and `Symbol.for`)
* Bigint (`wasm:js-bigint`):
    * Type test: `test`
    * (all other manipulations lacked motivation and were removed)

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

There is more discussion on this topic in [issue #6](https://github.com/WebAssembly/js-primitive-builtins/issues/6), although the original title and post were about more targeted concerns that have since been addressed.

## Open questions

### `externref` or `anyref`

As shown above, the typical use case for this proposal manipulates `anyref` as a universal representation.
Indeed, while we focus here on the primitives that JS must understand, most values in the universal representation remain Wasm `struct`s.

Yet, the builtins we propose here manipulate `externref`s, because the JS string builtins set `externref` as a precedent.

For consistency, `externref` is better.
But in pratice, it means toolchains will need to insert `any.convert_extern` and `extern.convert_any` after/before every call to the builtins presented here.
This has a code size cost, but maybe it will also have a run-time cost.

We expect actual experimentation in a (reasonably) optimizing compiler to settle the performance question.
If it turns out that there is no cost to using `externref` over `anyref`, we should probably stick with `externref`.
Otherwise, we may want to reconsider during experimentation.

See [#27](https://github.com/WebAssembly/js-primitive-builtins/issues/27) for a discussion.

### Symbol equality

We introduce a `"wasm:js-symbol" "equals"` to test equality of symbol primitives.
We do this by analogy with `"wasm:js-string" "equals"`, which was already specified as part of the JS String Builtins proposal.

However, it has been argued that is nothing but a special-case of `Object.is`, which is importable as is.
The same argument could have been made about `"wasm:js-string" "equals"`, though, and yet, it was included.

So it is unclear what to do.
Should we have symbol equality for consistency with strings?
Or should we remove it on the grounds that it can be done with `Object.is`?

Was there a definite performance advantage to `"wasm:js-string" "equals"`, owing to knowledge that *only* strings need to be handled?
If yes, that would support doing the same for symbols.

See [#1](https://github.com/WebAssembly/js-primitive-builtins/issues/1) for more context.

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

In general, languages don't necessarily agree on the exact string coming out of `binary64`-to-string conversion.
They may disagree on

* the scale at which to switch from fixed to scientific notation, and
* how many trailing 0's to display (including whether to force a `.` to display a trailing 0 in the 10^(-1) position).

However, there is strong alignment on a) how many non-zero digits to emit, and b) what those non-zero digits must be.

If required, it is possible to make the necessary adjustments in a post-processing step, which does not require big tables nor fancy tricks.
That said, languages targeting JavaScript environments tend to accept those small differences in exchange for the native conversion.

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

Parsing `bigint`s is similarly possible by importing `BigInt` and calling it with a `string` argument.
That one *throws* a JS exception (a `SyntaxError`) if the argument is not a valid big integer.
This is unusual for Wasm, but is not unheard of.
In fact, it may be useful for the surrounding Wasm code to catch that exception and turn it into a language-dependent signal, as it is annoying to test the validity ahead of time.

### JS operators `x % y` and `x | 0`

These operators may receive (partial) support from dedicated hardward instructions: `fprem` for `x % y`, and the ARM-specific `FJCVTZS` for `x | 0`.
They could therefore be useful as builtins.
However, if we wanted to expose them to Wasm, actual Wasm opcodes would probably be a better avenue.

### `toString` with radix for `i32` and `i64`

JavaScript provides conversion of number values to string with a specific radix, through `Number.prototype.toString`.
For example, `x.toString(16)`.
It is not importable as is, since it is a prototype method that requires access to the `this` value.

We considered adding a builtin for that.
V8 already contains dedicated code to recognize it, when hidden behind a bound `Function.prototype.call`.
That suggests that there is an incentive to support conversion of `i32` to string with radix as a builtin.

However, JS does not have support for converting `bigint`s with a specific radix.
Since our `i64` operations rely on `bigint` support, it means we cannot provide direct support for converting an `i64` with a radix.
For consistency, it seems best to leave the `i32` variant off the table as well.

Other than base 10, the only common bases are powers of 2 (namely 2, 8 and 16).
The latter are easy to efficiently implement in user-space, and the former is supported by `"fromI32"` and `"fromI64"`.
Therefore it should not be a real limitation.

### More on Symbols

Creation of symbols is achieved with the functions `Symbol` and `Symbol.for`.
These can be imported as is.

Extraction of the key of a symbol is, on a naive look similar: `Symbol.keyFor` can be imported as is.
However, it returns `undefined` when the given symbol has no associated key.
`undefined` can be tested after the fact with `"wasm:js-undefined" "test"`.

A case could be made to provide a builtin to extract the `[[Description]]` of a symbol.
In JavaScript, we do this with the accessor `Symbol.prototype.description`.
However, compared to the rest of `"wasm:js-symbol"`, it looks out of place.
It is unlikely that getting the description of a symbol would be on a performance-sensitive path, so we leave it out.

### `bigint` operations

Initially, this proposal considered operations on `bigint`s (besides the type test).
The following operations were considered:

* Bigint (`wasm:js-bigint`):
    * Create from primitive: `fromF64`, `fromI64`, `fromU64`
    * Extract to primitive: `convertToF64`, `wrapToI64`
    * Operations: `add`, `pow`, `shl`, etc. (the ones corresponding to JS operators)
    * Conversion to string: `toString`

This was based on the assumption that some toolchains would use them in hot paths, but that was apparently a misunderstanding (see [#24](https://github.com/WebAssembly/js-primitive-builtins/issues/24)), so we removed them.
