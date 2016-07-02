Silent Property Access on null/undefined for ECMAScript
-------------------------------------------------------

TL;DR: Relax the rules to return `undefined` for property access on `null` or `undefined` instead of throwing.

This is going to be a very controversial proposal but if it works it would be a very elegant solution to a significant problem.

### The Problem

How do you access deep properties into an object where intermediates may be null or undefined?

Currently the syntax is too bloated:

```js
let abc = a == null || a.b == null || a.b.c == null ? null : a.b.c;
```

With longer property names this becomes very messy to the point that library helpers keep being reinvented to minimize the boilerplate.

It is also a very common operation in JavaScript applications because they're often just JSON to UI processors.

### Examples

The proposed semantics is that property access on `null` or `undefined` returns `undefined`. Any other __ToObject__ operation still throws.

```js
let a = null;
let abc = a.b.c; // undefined
```

```js
let a = undefined;
let abc = a.b.c; // undefined
```

```js
let a = { b: null };
let abc = a.b.c; // undefined
```

```js
let a = { b: { c: null } };
let abc = a.b.c; // null
```

```js
let a = { b: null };
delete a.b.c; // throws a TypeError
```

### Specification

The specification simply changes the runtime semantics for property access and returns `undefined` in the case the baseValue is `null` or `undefined`.

_MemberExpression_ : _MemberExpression_ __[__ _Expression_ __]__

1. Let baseReference be the result of evaluating MemberExpression.
2. Let baseValue be ? GetValue(baseReference).
3. Let propertyNameReference be the result of evaluating Expression.
4. Let propertyNameValue be ? GetValue(propertyNameReference).
5. Let propertyKey be ? ToPropertyKey(propertyNameValue).
6. If the code matched by the syntactic production that is being evaluated is strict mode code, let strict be true, else let strict be false.
7. __If baseValue is `null` or `undefined`, return a value of type Reference whose base value component is `undefined`, whose referenced name component is `propertyKey`, and whose strict reference flag is `strict`.__
8. Return a value of type Reference whose base value component is baseValue, whose referenced name component is propertyKey, and whose strict reference flag is strict.

_MemberExpression_ : _MemberExpression_ __.__ IdentifierName

1. Let baseReference be the result of evaluating MemberExpression.
2. Let baseValue be ? GetValue(baseReference).
3. Let propertyNameString be StringValue of IdentifierName.
4. If the code matched by the syntactic production that is being evaluated is strict mode code, let strict be true, else let strict be false.
5. __If baseValue is `null` or `undefined`, return a value of type Reference whose base value component is `undefined`, whose referenced name component is propertyNameString, and whose strict reference flag is `strict`.__
6. Return a value of type Reference whose base value component is baseValue, whose referenced name component is `propertyNameString`, and whose strict reference flag is strict.

### Known Issues

#### Early errors are nice

The general objection to this is that it won't catch errors early enough. This is an area where I feel like the ship has already sailed for JavaScript. Generally the language is very permissive. Even without this `undefined` values travel around code through calls like `a.b` and turn into `NaN` freely. Generally this is solved with additional tooling on top of the runtime semantics such as linters.

If you truly want to protect against this then adding strong typing through dynamic runtime checks or static typing through static type systems like TypeScript or Flow is going to cover a lot more than just this one special case.

#### Why the special case?

These are the only types that currently DOES throw so this is actually getting rid of a special case. `boolean`, `number`, `string` and `symbol` all doesn't throw. It's no weirder that `123.foo()` not throwing.

`Object.assign` and many others already special case `null` and `undefined` to be treated roughly like an empty object anyway because it is so convenient.

#### Is this change web safe and secure?

I don't know. It is plausible that things on the web relies on an error being thrown. Either accidentally or intentionally as a security measure. It would have to be tested and evaluated.

Usually relaxing errors is something we let ourselves do but this might be frequent enough that it might cause problems.

#### No explicit syntax for communicating intent

Unlike other proposals this would come with a way to explicitly communicating the intent to access property on an object that might be null.

For example, type systems generally want to warn you if you access a missing property. It can't do that if this is a legit use case.

This is already a problem for the `a.b` case where `b` may or may not exist on `a` if `a` is a union type.

JavaScript doesn't generally add syntax for the purpose of supporting type systems neither. It's up to the type systems to propose additional syntax for communicating this intent if you use them.

### Why not existential or null propagator operator?

There have been long lived proposals to support an explicit operator to do this:

```js
let abc = a?.b?.c;
```

CoffeeScript, C#, and others have this.

I don't really have any concrete reason why this wouldn't be a good idea, but I'd like to explore an alternative and see if that is possible.
