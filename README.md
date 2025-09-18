## Problem

People used to extend the prototype but this isn't recommended anymore, now what?

## Proposal

Extension functions are a simple solution to this and are already implemented in multiple other languages like Kotlin, Swift and C#.

```js
function countWords() {
  return this.split(/\s/).length;
}

"Hello, World!".countWords();
// Returns: 2
```

## Specification details

### Only available in modules

Extension functions are basically simple: if a function is in the current scope, and you call it with a receiver, then it acts as an extension function.

This is potentially dangerous when the scope is too broad, it could accidentaly make calls to functions available in the global scope. This is why this feature is limited to modules.

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

### Extension functions are normal functions, they only differ on call

We can already do `someMethod.bind(obj)` everywhere, this proposal is essentially a syntactic sugar.

### Extension functions cannot override class or object properties

```js
function toSorted() {
  return this.toSorted((a, b) => a - b);
}

[1, 30, 4, 21, 100000].toSorted();
// Returns: [1, 100000, 21, 30, 4]
// Reason is `Array.prototype.toSorted()` has been called instead of the extension function
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
