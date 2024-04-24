# Write-Once `const` Declarations

Author: Nicholas C. Zakas
Stage: -1 (not yet submitted)

This proposal seeks to allow `const` declarations to be uninitialized and later written to just once.

You can browse the [ecmarkup output](https://nzakas.github.io/proposal-write-once-const/)
or browse the [source](https://github.com/nzakas/proposal-write-once-const/blob/HEAD/spec.emu).

## Motivation

In certain situations, a constant binding's value may not be calculable at the time of declaration, leaving developers to use `let` even though the binding's value will never change once set. Using `let` on bindings that should be immutable can lead to errors. 

### Use Case #1: Multiple conditions to calculate a value

This often happens when the value of a binding needs more than two conditions to be evaluated in order to determine the correct value to use. For example:

```js
let value;

if (someCondition) {
  value = 1;
} else if (someOtherCondition) {
  value = 2;
} else {
  value = 3;
}
```

This can be rewritten with a nested ternary, like this:

```js
const value = someCondition
  ? 1
  : otherCondition
  ? 2
  : 3;
```

While some might argue that this solves the problem, it is debatable how readable this code is.

### Use Case #2: Assigning to Multiple Bindings

The problem becomes more apparent when you need to set multiple binding values using the same conditions, as in this example:

```js
let value1, value2;

if (someCondition) {
  value1 = 1;
  value2 = 5;
} else if (someOtherCondition) {
  value1 = 2;
  value2 = 10;
} else {
  value1 = 3;
  value2 = 15;
}
```

In this case, you have no option to use nested ternaries without turning to destructuring and temporary object creation, such as:

```js
const  { value1, value2 } = someCondition
  ? { value1: 1, value2: 5 }
  : someOtherCondition
  ? { value1: 2, value2: 10 }
  : { value1: 3, value2: 15 };
```

This example further obfuscates the actual logic, making it more difficult to understand the purpose of the code all in the service of creating an immutable binding.

Some real world examples of this:

* **Faker:** [`month()`](https://github.com/faker-js/faker/blob/16ba43a6a4d1c93ac588c6b4c20b8c2a40213bdb/src/modules/date/index.ts#L623C9-L633)
* **Faker:** [`fakeEval()`](https://github.com/faker-js/faker/blob/16ba43a6a4d1c93ac588c6b4c20b8c2a40213bdb/src/modules/helpers/eval.ts#L82C9-L89)
* **Faker:** [`networkInterface()`](https://github.com/faker-js/faker/blob/16ba43a6a4d1c93ac588c6b4c20b8c2a40213bdb/src/modules/system/index.ts#L265C9-L295)

### Use Case #3: `try-catch`-initialized Bindings

In some cases, you may want to initialize a binding inside of a `try-catch` but allow that binding to be accessed outside of the `try-catch`. For example:

```js
async function doSomething() {

  let result;

  try {
      result = await someOperationThatMightFail();
  } catch (error) {
    
      // handle error and...
      return;
  }

  doSomethingWith(result);
}
```

Here, the `result` binding needs to be defined using `let` even though it is only ever read after being set.

Some real world examples of this:

* **Vite:** [Calculating cached meta data](https://github.com/vitejs/vite/blob/69773520f214027070b0ff1a3344394f37ef19f8/packages/vite/src/node/optimizer/index.ts#L355-L377)
* **Vite:** [`init()`](https://github.com/vitejs/vite/blob/69773520f214027070b0ff1a3344394f37ef19f8/packages/create-vite/src/index.ts#L258C7-L366)
* **node-pg-migrate:** [tsconfig calculation](https://github.com/salsita/node-pg-migrate/blob/fde10af20cdaf822fc9de49c71c9124f09fd6b71/bin/node-pg-migrate.ts#L253-L282)

## Proposal

I propose that `const` declarations no longer require initialization. Specifically:

1. An uninitialized `const` declaration is no longer a syntax error.
1. Attempting to read an uninitialized `const` binding before its value is set causes `ReferenceError: Cannot access 'name' before initialization.`. (Differs from `let`, which allows you to check the value before initialization.)
1. An uninitialized `const` binding may have its value set exactly once.
1. Attempting to set the value of an uninitialized `const` binding more than once causes the same `TypeError: Assignment to constant variable.` as assigning to any other `const` binding.
1. Using `typeof` on an uninitialized `const` binding returns `"undefined"`

For example:

```js
const value;

if (someCondition) {
  value = 1;
} else if (someOtherCondition) {
  value = 2;
} else {
  value = 3;
}

value = 4;    // TypeError: Assignment to constant variable.

const value2;
console.log(typeof value2);   // "undefined"
console.log(value2);  // ReferenceError: Cannot access 'value2' before initialization.
```

## Similar Immutable Bindings in Other Languages

Other languages supporting write-once immutable bindings typically have the following behavior:

1. The binding cannot be read until a value is set (this is an error)
1. The binding becomes immutable upon assignment of a value (any attempt to change the value is an error)

The following languages support creating immutable bindings in this way.

### Rust

Rust allows the declaration of immutable bindings through the use of the `let` keyword. The following example comes from the [Rust Book](https://doc.rust-lang.org/rust-by-example/variable_bindings/declare.html):

```rs
fn main() {
    // Declare a variable binding
    let a_binding;

    {
        let x = 2;

        // Initialize the binding
        a_binding = x * x;
    }

    println!("a binding: {}", a_binding);

    let another_binding;

    // Error! Use of uninitialized binding
    println!("another binding: {}", another_binding);
    // FIXME ^ Comment out this line

    another_binding = 1;

    println!("another binding: {}", another_binding);
}
```

### Swift

Swift uses an immutable `let` declaration for defining constants. Here's an example from the [Swift Book](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/thebasics/#Constants-and-Variables):

```swift
var environment = "development"
let maximumNumberOfLoginAttempts: Int
// maximumNumberOfLoginAttempts has no value yet.


if environment == "development" {
    maximumNumberOfLoginAttempts = 100
} else {
    maximumNumberOfLoginAttempts = 10
}
// Now maximumNumberOfLoginAttempts has a value, and can be read.
```

### Java

Java has a similar feature in the form of blank finals. A blank final uses the `final` keyword to indicate that the value of a binding must not change. That binding need not be initialized at declaration and cannot be changed once a value is set. Here's an example from [Wikipedia](https://en.wikipedia.org/wiki/Final_%28Java%29#Blank_final):

```java
final boolean isEven;

if (number % 2 == 0) {
  isEven = true;
}

System.out.println(isEven); // compile-error because the variable was not assigned in the else-case.
```

Java also checks code paths at compile-time to ensure that a blank final's value cannot possibly be set more than once.

## Comparison to Various Expression-Based Solutions

One of the early pieces of feedback was, "don't expression statements solve these use cases better?" In some cases that answer is yes, but I don't think that necessarily means that write-once `const` can't be a part of toolkit for developers to address these problems.

### Use Case #1: Multiple conditions to calculate a value

Here is the approach with write-once `const`:

```js
const value;

if (someCondition) {
  value = 1;
} else if (someOtherCondition) {
  value = 2;
} else {
  value = 3;
}
```

Here is the approach with an imaginary `if` expression:

```js
const value = if (someCondition) {
  1;
} else if (otherCondition) {
  2;
} else {
  3;
}
```

Here is an approach with a proposed `do` expression:

```js
const value = do {
  if (someCondition) {
    1;
  } else if (otherCondition) {
    2;
  } else {
    3;
  }
}
```

While `if` and `do` expressions would solve this problem, there is room to debate which option is more readable.

If JavaScript ends up with a Rust-style `match` expression, then I concede that is the best solution for this use case:

```js
const value = match {
  when someCondition: 1;
  when otherCondition: 2;
  default: 3;
}
```

### Use Case #2: Assigning to Multiple Bindings

Here is the approach with write-once `const`:

```js
const value1, value2;

if (someCondition) {
  value1 = 1;
  value2 = 5;
} else if (someOtherCondition) {
  value1 = 2;
  value2 = 10;
} else {
  value1 = 3;
  value2 = 15;
}
```

Here is the approach with an imaginary `if` expression:

```js
const { value1, value2 } = if (someCondition) {
  ({ value1: 1, value2: 5 });
} else if (someOtherCondition) {
  ({ value1: 2, value2: 10 });
} else {
  ({ value1: 3, value2: 15 });
}
```

Here is an approach with a proposed `do` expression:

```js
const { value1, value2 } = do {
  if (someCondition) {
    ({ value1: 1, value2: 5 });
  } else if (someOtherCondition) {
    ({ value1: 2, value2: 10 });
  } else {
    ({ value1: 3, value2: 15 });
  }
}
```

The advantage of write-once `const` in this scenario is the avoidance of creating temporary objects in order to initialize multiple variables.

### Use Case #3: `try-catch`-initialized Bindings

Here is the approach with write-once `const`:

```js
async function doSomething() {

  const result;

  try {
      result = await someOperationThatMightFail();
  } catch (error) {
    
      // handle error and...
      return;
  }

  doSomethingWith(result);
}
```

Here is the approach with an imaginary `try` expression:

```js
async function doSomething() {

  const result = try {
      await someOperationThatMightFail();
  } catch (error) {
    
      // handle error and...
      return;
  }

  doSomethingWith(result);
}
```

Here is an approach with a proposed `do` expression:

```js
async function doSomething() {

  const result = do {
    try {
      await someOperationThatMightFail();
    } catch (error) {
    
      // handle error and...
      return;
    }
  }

  doSomethingWith(result);
}
```

For this use case, I think the `try` expression is what I'd prefer as a developer overall as it has the clearest indication of what is happening. I'd put write-once `const` in second place in this list.

## Frequently Asked Questions

### Why throw an error when the uninitialized binding is read?

There are two primary reasons:

1. This behavior is aligned with how uninitialized lexical bindings already work in JavaScript. 
1. This is how write-once bindings are implemented in Rust, Swift, and Java. 

However, I have no particularly affinity for this behavior other than consistency with existing behavior.

(Another option would be to treat an uninitialized `const` binding the same as an uninitialized `let` binding, in which the value is treated as `undefined` in all respects and does not throw an error when the value is ready before initialization.)

### Why propose this when people seem to be clamoring for expression statements?

Write-once `const` is significally smaller in scope to solve similar problems. My hope is that, if accepted, this proposal could move more quickly through to acceptance. For reference:

* [`do` Expressions](https://github.com/tc39/proposal-do-expressions) (stage 1) were first proposed in 2018 and hasn't had an update in three years.
* [Pattern Matching](https://github.com/tc39/proposal-pattern-matching) (stage 1) was first proposed in 2020 and last presented in 2022.

Plus, Rust and Swift support both write-once bindings and `if`, `switch`, etc. exprssions -- it seems that there's plenty of room for both solutions in a language.

## Why didn't you show using functions to address these use cases?

All of the use cases can definitely be rewritten to use functions that encapsulate the logic, however, that's not what developers are actually doing. They're using `let` because they can't use `const`, and they're not wanting to extract that logic outside of the current function to do so.

## Maintain your proposal repo

  1. Make your changes to `spec.emu` (ecmarkup uses HTML syntax, but is not HTML, so I strongly suggest not naming it “.html”)
  1. Any commit that makes meaningful changes to the spec, should run `npm run build` to verify that the build will succeed and the output looks as expected.
  1. Whenever you update `ecmarkup`, run `npm run build` to verify that the build will succeed and the output looks as expected.

  [explainer]: https://github.com/tc39/how-we-work/blob/HEAD/explainer.md
