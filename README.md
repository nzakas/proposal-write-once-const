# Write-Once `const` Declarations

Author: Nicholas C. Zakas

This proposal seeks to allow `const` declarations to be uninitialized and later written to just once.

You can browse the [ecmarkup output](https://nzakas.github.io/proposal-write-once-const/)
or browse the [source](https://github.com/nzakas/proposal-write-once-const/blob/HEAD/spec.emu).

## Motivation

In certain situations, a constant binding's value may not be calculable at the time of declaration, leaving developers to use `let` even though the binding's value will never change once set. Using `let` on bindings that should be immutable can lead to errors. This often happens when the value of a binding needs more than two conditions to be evaluated in order to determine the correct value to use. For example:

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

### Assigning to Multiple Bindings

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

### `try-catch`-initialized Bindings

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

We propose that `const` declarations no longer require initialization. Specifically:

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

## Frequently Asked Questions

**Why throw an error when the uninitialized binding is read?**

This seems to be the convention in other languages with similar features. Another option is to treat the binding just like a `let` binding that hasn't been initialized, so that people can test to see if it's equal to `undefined`. However, I believe the use of `typeof` is a better option for that use case.

## Maintain your proposal repo

  1. Make your changes to `spec.emu` (ecmarkup uses HTML syntax, but is not HTML, so I strongly suggest not naming it “.html”)
  1. Any commit that makes meaningful changes to the spec, should run `npm run build` to verify that the build will succeed and the output looks as expected.
  1. Whenever you update `ecmarkup`, run `npm run build` to verify that the build will succeed and the output looks as expected.

  [explainer]: https://github.com/tc39/how-we-work/blob/HEAD/explainer.md
