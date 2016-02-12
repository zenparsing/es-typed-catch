## Typed Catch Blocks for ECMAScript

### Motivation

ECMAScript currently only supports a single unconditional catch clause.  If the programmer only wants to handle certain errors, they must test the exception value within the catch clause and potentially rethrow the error.  Users frequently filter for errors based on type using the `instanceof` operator.

```js
try {

    run();

} catch (error) {

    // Test for an error type
    if (error instanceof SomeError) {
        console.log("Error handled - it's OK.")
    } else {
        // We can't handle this error, so rethrow it
        throw error;
    }
}
```

With the introduction of async functions, errors which are currently propagated through ad hoc asynchronous mechanisms such as callback arguments or events will now be propagated using `try-catch` exception handling.  As a result, users will want a more ergonomic way to accomplish targeted exception handling.

### Overview

In addition to the current unconditional catch clause, we allow a list of typed catch clauses.  Each typed catch clause contains a type specifier.  When an exception is propagated from a try block, we apply the `instanceof` operator to each type specifier in order associated with that try block.

```js
try {

    somethingDangerous();

} catch (error : TypeError) {

    // This will catch a TypeError

} catch (error : SyntaxError) {

    // This will catch a SyntaxError

} catch (error : Error) {

    // This will catch an Error (if the previous catch clauses didn't match)
}
```

The preceding example is equivalent to the following:

```js
try {

    somethingDangerous();

} catch (error) {

    let _error = Object(error);

    if (_error instanceof TypeError) {

        // This will catch a TypeError

    } else if (_error instanceof SyntaxError) {

        // This will catch a SyntaxError

    } else if (_error instanceof Error) {

        // This will catch an Error (if the previous catch clauses didn't match)

    } else {

        throw error;
    }
}
```

### What about realms?

The `instanceof` operator will typically evaluate to false when the operands are from different realms (i.e. iframes):

```js
let a = new iframe.contentWindow.Array();
assert(a instanceof Array === false);
assert(Array.isArray(a) === true);
```

Because catch clauses are matched using `instanceof`, this could potentially lead to surprising behavior for the built-in error classes.

```js
try {
    throw new iframe.contentWindow.Error("Error from an iframe");
} catch (error : Error) {
    // This block doesn't run
} catch {
    // But this block does
}
```

ES6 introduces `Symbol.hasInstance`, which can be used to modify the behavior of the `instanceof` operator.  Using this protocol, we can customize the matching behavior of catch clauses to support cross-frame error handling:

```js
const CrossFrameError = {
    [Symbol.hasInstance](x) {
        return Object.prototype.toString.call(x) === "[object Error]";
    }
};

try {
    throw new iframe.contentWindow.Error("Error from an iframe");
} catch (error : CrossFrameError) {
    // This block will run
}
```

### What about multiple library versions?

In both browser and server environments, it's commonly the case that multiple versions of the same library are simultaneously loaded.  In some cases, multiple instances of the same library may be loaded.  This can present challenges for nominal type checking.

```js
// Different versions of the same library
import * as A from "a/shell-tools";
import * as B from "b/shell-tools";

try {
    throw A.ShellError();
} catch (error : B.ShellError) {
    // This block won't run
}
```

User-defined error classes can support a more coarse-grained type check by providing a custom implementation of the `Symbol.hasInstance` class method.

```js
const errorBrand = Symbol.for("shell-tools.ShellError");

export class ShellError extends Error {
    get [errorBrand]() { return true; }
    static [Symbol.hasInstance](x) { return x[errorBrand] === true; }
}
```

### What about pattern matching?

Pattern matching relies upon *refutable* patterns.  A refutable pattern is a pattern that can fail to match.  It's tempting to think of `catch` in terms of refutable matching:  we try a series of patterns, and we execute the block associated with the first matching pattern.  However, there are some problems:

- The current catch clause accepts *irrefutable* destructuring patterns.  If we wanted a pattern matching `catch` we could not place those patterns in the `catch` head.
- Users typically don't want to match errors based on structural typing.  The most common way to match errors is by error type.  Pattern matching could support `if` predicates, but putting a type check into an `if` predicate is unnecessarily verbose.
- In the context of Javascript, pattern matching will probably be a tool used by experienced programmers.  In contrast, we want targeted exception handling to be natural and concise for users at all experience levels.

### Syntax

```
TryStatement:
    `try` Block CatchClauses
    `try` Block Finally
    `try` Block CatchClauses Finally

CatchClauses:
    DefaultCatch
    CatchList DefaultCatch

CatchList:
    Catch
    CatchList Catch

Catch:
    `catch` `(` CatchParameter `:` LeftHandSideExpression `)` Block

DefaultCatch:
    `catch` `(` CatchParameter `)` Block
```
