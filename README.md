# Write-Once `const` Declarations

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

## Proposal

We propose that `const` declarations no longer require initialization. Specifically:

1. An uninitialized `const` declaration is no longer a syntax error.
1. Attempting to read an uninitialized `const` binding before its value is set causes `ReferenceError: Cannot access 'name' before initialization.`. (Differs from `let`, which allows you to check the value before initialization.)
1. An uninitialized `const` binding may have its value set exactly once.
1. Attempting to set the value of an uninitialized `const` binding more than once causes the same `TypeError: Assignment to constant variable.` as assigning to any other `const` binding.

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
console.log(value2);  // ReferenceError: Cannot access 'value2' before initialization.
```

## Other Languages

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


## Maintain your proposal repo

  1. Make your changes to `spec.emu` (ecmarkup uses HTML syntax, but is not HTML, so I strongly suggest not naming it “.html”)
  1. Any commit that makes meaningful changes to the spec, should run `npm run build` to verify that the build will succeed and the output looks as expected.
  1. Whenever you update `ecmarkup`, run `npm run build` to verify that the build will succeed and the output looks as expected.

  [explainer]: https://github.com/tc39/how-we-work/blob/HEAD/explainer.md
