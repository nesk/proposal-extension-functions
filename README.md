## Problem

Applying multiple transformations to a value produces code that isn't easy to read in ECMAScript:

```js
some(custom(transformations(on(string))))
```

This code doesn't execute like a human would read it and actually runs in this order:

```
string -> on -> transformations -> custom -> some
```

The ECMAScript community has been searching for a lot of ways to write code that's easy to read, multiple proposals have been submitted.

## Proposal

Extension functions are a simple solution to this and are already implemented in multiple other languages like [Kotlin](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions/), [Swift](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions/) and [C#](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods).

```js
function countWords() {
  return this.split(/\s/).length;
}

"Hello, World!".countWords();
// Returns: 2
```

There is no need to create new functions if a library author wants to add support for extension functions to its already existing library:

```js
function countWords(str) {
  const self = this === globalThis ? str : this;
  return self.split(/\s/).length;
}

countWords("Hello, World!"); // Old API is still supported
"Hello, World!".countWords(); // Extension API works by using the same function
```

## Specification details

### Only available in modules

Extension functions are basically simple: if a function is in the current scope, and you call it with a receiver, then it acts as an extension function.

This feature is limited to modules because it's potentially dangerous when the scope is too broad, it could accidentaly make calls to functions available in the global scope.

#### Global functions can't be extension functions

It also means that, even in a module, a global function cannot be used as an extension function:

```html
<script>
  function countWords() {
    return this.split(/\s/).length;
  }
</script>
<script type="module">
  "Hello, World!".countWords();
  // Throws: Uncaught TypeError: "Hello, World!".countWords is not a function
</script>
```

### Extension functions are normal functions

An extension function is a normal function but called with a receiver.

This is something we can already do by using `Function.prototype.bind()`, `Function.prototype.call()` or `Function.prototype.apply()`, this proposal is essentially a syntactic sugar.

```js
// This code
"Hello, World!".countWords();

// Is syntactic sugar for
countWords.call("Hello, World!");
```

### Extension functions cannot override class or object properties

```js
function toSorted() {
  return this.toSorted((a, b) => a - b);
}

[1, 30, 4, 21, 100000].toSorted();
// Returns: [1, 100000, 21, 30, 4]
// If our custom function had been called, it would have return [1, 4, 21, 30, 100000]
// Since `Array.prototype.toSorted()` already exists, it has been called instead
```

```js
const greetings = {
  hello() {
    return "Hello, World!";
  }
}

function hello() {
  return "Hello, dear reader.";
}

greetings.hello();
// Returns: "Hello, World!"
```

Just like in [Kotlin](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions/):

> If a class has a member function, and an extension function is defined which has the same receiver type, the same name, and is applicable to given arguments, the **member always wins.**

[Swift](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions/):

> Extensions can add new functionality to a type, but they can’t override existing functionality.

and [C#](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods#binding-extension-members-at-compile-time):

> You can use extension members to extend a class or interface, but not to override behavior defined in a class. An extension member with the same name and signature as an interface or class members are never called.

### Extension functions cannot be called dynamically

This is not possible:

```js
function countWords() {
  return this.split(/\s/).length;
}

"Hello, World!"['countWords']();
// Throws: Uncaught TypeError: "Hello, World!".countWords is not a function
```

### Only functions can be used as extensions

Getter and setters are out of scope for this proposal.

Only variables with `typeof === 'function'` can be used as extensions:

```js
const someIndex = 2;
['a', 'b', 'c'].someIndex
// Returns: undefined
```
