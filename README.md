# FlowSure has been integrated in Cause & Effect

See: https://github.com/zeixcom/cause-effect

While signal based reactivity is not the same as functional pipelines – what flow sure stood for – most of the paradigms of this exploration have been integrated in Cause & Effect:

- Type-safe, non-nullable values as a guarantee throughout the signal graph, with guards at the entry point (`State` signals)
- Memoized computations (`Memo` signals) that can be composed to a reactive graph are more flexible than linear pipelines
- `Task` signals incorporate the seamless integration of async computations we originally strived for with `task()`
- The `match()` function for `Effect` in Cause & Effect is designed after the `.match()` method in this project

**This repository will not be updated anymore.** Use Cause & Effect instead.

Cause & Effect features one of the fastest signal engines and contains a complete Set of primitives that give you all the guarantees lacking in imperative programming. It offers fine-grained reactivity for composite signals (`Store`, `List`, derived `Collection`), and primitives for deriving states from event streams (`Sensor`) and property descriptors with swappable backing signals (`Slot`).

---

# FlowSure

Version 0.10.0

Inspired by functional programming, **FlowSure** provides tools like `ok()`, `maybe()`, `result()`, `task()` and `flow()` to simplify complex workflows. Here's a quick example:

```js
import { ok, result } from "@efflore/flow-sure";

ok(5)
    .map(x => x * 2)
    .filter(x => x > 5)
    .match({
        Ok: value => console.log("Success:", value),
        Nil: () => console.warn("No value passed the filter"),
        Err: error => console.error("Error:", error.message),
    });

result(() => JSON.parse("invalid json"))
    .match({
        Ok: value => console.log("Parsed:", value),
        Err: error => console.error("Failed to parse:", error.message),
    });
```

## Key Features

- **Simplify Error Handling:** Capture and propagate errors elegantly with `Result` types.
- **Composable Flows:** Chain both sync and async functions seamlessly using monadic methods.
- **Declarative Control:** Use `flow()` to build clear, predictable pipelines.
- **Immutability Assurance:** Safely wrap and clone mutable objects to avoid unintended side effects.

**FlowSure** is very lightweight: around 1kB gzipped.

## Installation

```bash
# with npm
npm install @efflore/flow-sure

# or with bun
bun add @efflore/flow-sure
```

## Basic Usage

### Monadic Control with Result Types

FlowSure's `Result` types (`Ok`, `Err`, `Nil`) let you manage value transformations and error handling with `.map()`, `.chain()`, `.filter()`, `.guard()`, `.or()` and `.catch()` methods. The `match()` method allows for pattern matching according to the `Result`type. Finally, `get()` will return the value (if `Ok`) or `undefined` (if `Nil`) or rethow the catched `Error` (if `Err`).

```js
import { ok } from "@efflore/flow-sure";

ok(5).map(x => x * 2).filter(x => x > 5).match({
    Ok: value => console.log("Transformed Value:", value),
    Nil: () => console.warn("Value didn't meet filter criteria"),
    Err: error => console.error("Error:", error.message)
});
```

#### Monadic Methods Table

| Method      | `Ok<T>`       | `Nil`               | `Err<E extends Error>` | Argument Type                                           | Return Type                |
|-------------|---------------|---------------------|------------------------|---------------------------------------------------------|----------------------------|
| `.map()`    | **Yes**       | No-op               | No-op                  | `(value: T) => U`                                       | `Ok<U>`, `Nil` or `Err<E>` |
| `.chain()`  | **Yes**       | No-op               | No-op                  | `(value: T) => Result<U>`                               | `Result<U>`                |
| `.await()`  | **Yes**       | No-op               | No-op                  | `async (value: T) => Result<U>`                         | `Promise<Result<U>>`       |
| `.filter()` | **Yes**       | No-op               | Converts to `Nil`      | `(value: T) => boolean`                                 | `Maybe<T>`                 |
| `.guard()`  | **Yes**       | No-op               | Converts to `Nil`      | `(value: T) => value is U`                              | `Maybe<T>`                 |
| `.or()`     | No-op         | **Yes**             | **Yes**                | `() => T \| undefined`                                  | `Ok<T>` or `Maybe<T>`      |
| `.catch()`  | No-op         | No-op               | **Yes**                | `(error: E) => Result<T>`                               | `Result<T>`                |
| `.match()`  | **Yes**       | **Yes**             | **Yes**                | `(value: T) => any`, `() => any` or `(error: E) => any` | `any`                      |
| `.get()`    | Returns value | Returns `undefined` | Throws error           |  --                                                     | `T`, `void` or `E`         |

#### Explanation of Each Method

* `.map()`: Transforms the value if it exists (`Ok`); `Nil` and `Err` remain unchanged.
* `.chain()`: Chains a function that returns a new `Result`, only applies to `Ok`.
* `.await()`: Chains an async function that returns a `Promise` for a new `Result`, only applies to `Ok`.
* `.filter()`: Filters `Ok` based on a condition; converts to `Nil` if the condition is not met.
* `.guard()`: Similar to `.filter()`, but specifically used for type narrowing on `Ok`.
* `.or()`: Provides a fallback for `Nil` and `Err`, leaving `Ok` unchanged.
* `.catch()`: Handles errors within `Err` types, leaving `Ok` and `Nil` unchanged.
* `.match()`: Allows pattern matching across `Ok`, `Nil`, and `Err`.
* `.get()`: Retrieves the contained value, returning `undefined` for `Nil` and throwing for `Err`.

### Handling Optional or Missing Values with maybe()

Using `maybe()` ensures that `undefined` or `null` values are handled explicitly, reducing the risk of runtime errors caused by forgotten null checks. Instead of `if` statements scattered throughout your code, you can use methods like `.map()`, `.filter()`, and `.match()` to express intent clearly.

```js
import { maybe } from "@efflore/flow-sure";

const optionalValue = undefined; // Could also be null or an actual value
maybe(optionalValue)
    .map(value => value * 2)
    .filter(value => value > 5)
    .match({
        Ok: value => console.log("Value:", value),
        Nil: () => console.warn("Value is either missing or didn't meet criteria")
    });
```

### Handling Exceptions with result()

`result()` is especially useful for wrapping code that may throw unexpected errors, such as parsing user input or accessing properties on potentially null objects.

It captures exceptions and converts them into `Err` values, allowing you to handle errors gracefully within the chain.

```js
import { result } from "@efflore/flow-sure";

result(() => {
    // Function that may throw an error
    return JSON.parse("invalid json");
}).match({
    Ok: value => console.log("Parsed JSON:", value),
    Err: error => console.error("Failed to parse JSON:", error.message)
});
```

### Handling Promises with task()

Use `task()` to retrieve and handle a promised result, wrapping it in `Result` types (`Ok`, `Err`, `Nil`). Here's an example of how you can add retry logic for async operations:

```ts
import { task, err } from "@efflore/flow-sure";

const fetchData = async () => {
    const response = await fetch('/api/data');
    if (!response.ok) return err(`Failed to fetch data: ${response.statusText}`);
    return response.json();
}

const retry = <T>(
    fn: () => Promise<MaybeResult<T>>,
    retries: number,
    delay: number
) => task(fn).catch((error: Error) => {
        if (retries <= 0) return err(error);
        return new Promise(resolve => setTimeout(resolve, delay))
            .then(() => retry(fn, retries - 1, delay * 2));
    });

// 3 attempts, exponential backoff with initial 1000ms delay
const loadData = async () {
	const data = await retry(fetchData, 3, 1000);
	// Process data ...
}
```

### Using flow() for Declarative Control

`flow()` enables you to compose a series of functions (both sync and async) into a cohesive pipeline:

```js
import { flow } from "@efflore/flow-sure";

const processData = async () {
	const result = await flow(
		10,
		x => x * 2,
		async x => await someAsyncOperation(x)
	).match({
		Ok: finalValue => console.log("Result:", finalValue),
		Err: error => console.error("Error:", error.message)
	});
	// Render data ...
}
```

### Using ok() for Immutability Guarantees

The `ok()` function wraps values to enforce immutability and prevent multiple retrievals. For mutable objects, it attempts to clone the value using `structuredClone()`. This ensures that modifications to the original object do not affect the wrapped value.

> **Note:** Some types (e.g., DOM elements, promises, `WeakMap`, `WeakSet`) cannot be cloned due to `structuredClone()` limitations. In these cases, `ok()` falls back to treating the value as immutable without guarantees against external modification. Document these edge cases in your codebase if immutability is critical.

The value of an `Ok` instance can only be retrieved once using `.get()`. Any subsequent attempts will throw a `ReferenceError`. You can check if the value has already been consumed using `isGone()` or the `.gone` property on the instance.

```js
import { ok } from '@efflore/flow-sure';

// Example with a mutable object
const original = { a: 1, b: 2 };
const wrapped = ok(original);

// Modifying the original object does not affect the wrapped value
original.a = 42;
console.log(wrapped.get()); // { a: 1, b: 2 } (immutable clone)

// Attempting to retrieve the value again throws a ReferenceError
try {
    console.log(wrapped.get());
} catch (e) {
    console.error(e.message); // "Mutable reference has already been consumed"
}

// Check if the value has been consumed
console.log(wrapped.gone); // true
```

### Exported Helper Functions

**FlowSure** also exports the following utility functions it uses internally:

```ts
isFunction(value: unknown): value is (...args: any[]) => any
```
Checks if the given value is a function.

```ts
isAsyncFunction(value: unknown): value is (...args: any[]) => Promise<any>
```
Checks if the given value is an asynchronous function (returns a `Promise`).

```ts
isDefined(value: unknown): value is NonNullable<typeof value>
```
Checks if the given value is neither `null` nor `undefined`.

```ts
isMutable(value: unknown): value is Record<PropertyKey, unknown>
```
Checks if the given value is a mutable object (non-`null` and `typeof value === "object"`).

```ts
isInstanceOf<T>(type: new (...args: any[]) => T): (value: unknown) => value is T
```
Creates a type guard to check if a value is an instance of a specific class or type.

```ts
isError(value: unknown): value is Error
```
Checks if the given value is an Error instance.

```ts
log(msg: string, logger: (...args: any[]) => void = console.log): (...args: any[]) => any
```
Logs a message and additional arguments using the specified logger (default: console.log). Returns the first argument for chaining.

```ts
tryClone<T>(value: T, warn = true): T
```
Attempts to clone a mutable object using structuredClone(). If cloning fails, logs a warning (if warn is true) and returns the original value.

```ts
wrap<T>(value: MaybeResult<T>): Result<T>
```
Wraps a value in a `Result` container. If the value is an error, it returns `Err`. If the value is undefined or null, it returns `Nil`. Otherwise, it returns `Ok`. Values that are already of a `Result` type are not double-wrapped.

```ts
unwrap<T>(value: Result<T> | T | void): T | Error | void
```
Unwraps a `Result` container, returning the value if it is `Ok`, the error if it is `Err`, or undefined if it is `Nil`.
