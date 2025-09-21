# Extension functions proposal

Author: Johann Pardanaud

## Problem

Applying multiple transformations to a value produces code that isn't easy to read:

```js
transformations(custom(some(apply(string))));
```

This code doesn't execute like a human would read it and actually runs in this order:

```
string -> apply -> some -> custom -> transformations
```

The community has long been searching for solutions to write code that executes in the same order we read it as humans:

- Extending the prototype of native objects—[Mootools](https://mootools.net/core/docs/1.6.0/Class/Class#Class:implement) and other libraries did it—but is now deemed unfit due to prototype pollution.
- Providing wrappers that allow to chain methods, [like Lodash's `_.chain()` method.](https://lodash.com/docs/4.17.15#chain)
- Submitting proposals for [extension functions](https://github.com/tc39/proposal-extensions), [this-binding syntax](https://github.com/tc39/proposal-bind-operator), [pipe operator](https://github.com/tc39/proposal-pipeline-operator).
- Relying on transpilation step to extend the language, [like the bind operator plugin for Babel.](https://babeljs.io/docs/babel-plugin-proposal-function-bind)

## Extension functions

Extension functions are a solution to this and already implemented in many other languages like [Kotlin](https://kotlinlang.org/docs/extensions.html), [Swift](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions/) or [C#](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods).

```js
function countWords() {
  return this.split(/\s/).length;
}

// Any function within the current scope can be called with a receiver
"Hello, World!".countWords();
// Returns: 2
```

They allow to write code that is easy to read from left to right, here's a demonstration based on the first example:

```js
string.apply().some().custom().transformations();
```

They also are really appreciated in the community, based on [the votes and comments of an old TypeScript issue that wanted to introduce them.](https://github.com/microsoft/TypeScript/issues/9)

### What do they bring

Roman Elizarov—ex-project lead for Kotlin—[wrote an article about "Extension-oriented design"](https://elizarov.medium.com/extension-oriented-design-13f4f27deaee); he explains how this feature helps in extending the language for the developer needs:

> You are stuck with a vocabulary of operations on a class that designers of the original library had in mind. It cannot be extended by you. […]
>
> Modern languages […] solve this issue by supporting extension methods. You can add domain-specific extensions to the classes you do not control, so that your own function could be called in a way that resembles a call of a built-in method […]

He also argues how they can help architect your own classes by separating your core methods from the convenient ones, so your classes are easier to grasp.

Extension functions have a lot of use cases:

#### Write readable code with custom functions

```js
import { toURL, trimUTMParameters } from "./url-helpers";

const userSubmittedURL = "https://www.example.com/items?page=2&utm_content=buffercf3b2";

userSubmittedURL
  .toURL()
  .trimUTMParameters()
  .toString();
// Returns: "https://www.example.com/items?page=2"
```

#### Provide safe alternatives to static methods

Array grouping went from an instance method to a static one [due to web incompatibilities.](https://github.com/tc39/proposal-array-grouping?tab=readme-ov-file#why-static-methods) Instead of an instance method like `Array.prototype.groupBy()`, we went for a static method `Object.groupBy()`, which is totally understandable but impossible to chain.

Extension functions would allow developers to create their own custom functions to wrap static methods, enabling chaining:

```js
import { mapValues } from "object-helpers";

function groupBy(callbackFn) {
  return Object.groupBy(this, callbackFn);
}

function toJSON() {
  return JSON.stringify(this);
}

const inventory = [
  { name: "asparagus", type: "vegetables", quantity: 9 },
  { name: "bananas", type: "fruit", quantity: 5 },
  { name: "goat", type: "meat", quantity: 23 },
  { name: "cherries", type: "fruit", quantity: 12 },
  { name: "fish", type: "meat", quantity: 22 },
];

inventory
  .groupBy(item => item.type)
  .mapValues(items => items.map(item => item.name))
  .toJSON();
// Returns: '{"vegetables":["asparagus"],"fruit":["bananas","cherries"],"meat":["goat","fish"]}'
```

#### Existing functions can easily be adapted

There is no need to create new functions if a library author wants to add support for extension functions to its already existing library:

```js
function countWords(str) {
  const self = this === globalThis ? str : this;
  return self.split(/\s/).length;
}

// Old API is still supported
countWords("Hello, World!");

// Extension API works by using the same function
"Hello, World!".countWords();
```

## Specification details

### Only available in modules

Extension functions are basically simple: if a function is in the current scope, and you call it with a receiver, then it acts as an extension function.

This feature is limited to modules because it's potentially dangerous when the scope is too broad, it could accidentally make calls to functions available in the global scope.

Also, by limiting the scope, developers are able to better understand if a call is made to an extension function or to a member function.

### Global functions can't be extension functions

Since extension functions are limited to modules to avoid scope pollution, it also means that, even in a module, a global function cannot be used as an extension function:

```html
<script>
  function countWords() {
    return this.split(/\s/).length;
  }
</script>
<script type="module">
  "Hello, World!".countWords();
  // Throws: Uncaught TypeError: 'Hello, World!'.countWords is not a function
</script>
```

### Extension functions are normal functions

An extension function is a classic function—async or not—but called with a receiver.

This is something we can already do by using `Function.prototype.bind()`, `Function.prototype.call()` or `Function.prototype.apply()`, this proposal is essentially a syntactic sugar.

```js
// This code
"Hello, World!".countWords();

// Is syntactic sugar for
countWords.call("Hello, World!");
```

### Extension functions cannot override class or object properties

```js
function toString() {
  return "<REDACTED>";
}

const password = "5iadCWgh";
password.toString();
// Returns: "5iadCWgh"
```

```js
const greetings = {
  hello() {
    return "Hello, World!";
  },
};

function hello() {
  return "Hello, dear reader.";
}

greetings.hello();
// Returns: "Hello, World!"
```

Just like in [Kotlin](https://kotlinlang.org/docs/extensions.html):

> If a class has a member function, and an extension function is defined which has the same receiver type, the same name, and is applicable to given arguments, the member always wins.

[Swift](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions/):

> Extensions can add new functionality to a type, but they can’t override existing functionality.

and [C#](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods#binding-extension-members-at-compile-time):

> You can use extension members to extend a class or interface, but not to override behavior defined in a class. An extension member with the same name and signature as an interface or class members are never called.

A solution is to rename the function, which can be done when importing for example:

```js
import { toString as toRedactedString } from "security-helpers";

const password = "5iadCWgh";
password.toRedactedString();
// Returns: "<REDACTED>"
```

### Extension functions cannot be called dynamically

These are not possible:

```js
function countWords() {
  return this.split(/\s/).length;
}

const str = "Hello, World!"

str["countWords"]();
// Throws: Uncaught TypeError: str.countWords is not a function

str[countWords]();
// Throws: Uncaught TypeError: str[countWords] is not a function
```

### Only functions can be used as extensions

Getter and setters are out of scope for this proposal, mainly because they aren't available at top-level module.

Only variables with `typeof === 'function'` can be used as extensions:

```js
const someIndex = 2;
["a", "b", "c"].someIndex;
// Returns: undefined
```

### Nested properties can't be used as extension functions

The goal of this proposal is to allow developers to write code that's easy to read and similar to calling an instance method, therefor, it is impossible to use nested properties as extension functions:

```js
import * as helpers from "string-helpers";

const str = "Hello, World!";
str[helpers.countWords]();
// Throws: Uncaught TypeError: str[helpers.countWords] is not a function
```

Instead, the function should be extracted to a variable before using it as an extension:

```js
const { countWords } = helpers
str.countWords();
```

### Binding is out of scope

Binding functions like suggested in [tc39/proposal-bind-operator](https://github.com/tc39/proposal-bind-operator) is out of scope. The extension syntax requires parentheses after the function name, otherwise it will simply retrieve the value of the named property.

```js
function countWords() {
  return this.split(/\s/).length;
}

const bindedCountWords = "Hello, World!".countWords;
bindedCountWords();
// Throws: Uncaught TypeError: bindedCountWords is not a function
```

## Concerns to discuss

### Bundlers

Bundlers aggregate the modules into a single output file. If the latter isn't a module—and it generally isn't—then the extension functions will never be called.

A solution would be for bundlers to detect extension functions usage and add a simple `export {}` at the end of the output file. This implies that the output file is properly referenced in browsers with a `<script type="module">` or with an `.mjs` extension in alternative runtimes like Node.js.

Maybe limiting extension functions to modules is too conservative?
