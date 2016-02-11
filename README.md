## Typed Catch Expressions for ECMAScript

### Motivation

ECMAScript currently only support a single unconditional catch clause.  If the programmer only wants to handle certain errors, they must test the exception value within the catch clause and potentially rethrow the error.  Users frequently filter for errors based on type using the `instanceof` operator.

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

    if (error instanceof TypeError) {

        // This will catch a TypeError

    } else if (error instanceof SyntaxError) {

        // This will catch a SyntaxError

    } else if (error instanceof Error) {

        // This will catch an Error (if the previous catch clauses didn't match)

    } else {

        throw error;
    }
}
```

### What about realms and multiple library versions?


### What about pattern matching?


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
