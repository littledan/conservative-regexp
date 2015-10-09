# Partial revert of ES2015 RegExp semantics

## ES2015 makes regular expressions subclassable.

There are a couple things it changes vs ES5 which allow behavior to be overridden:

1. String functions which call things related to RegExps can be overridden using Symbol-named methods on RegExps, e.g., `RegExp.prototype[Symbol.match]`, `RegExp.prototype[Symbol.search]`, etc.
2. The RegExp backends to `String.prototype.split`, `String.prototype.match`, `String.prototype.search` and `String.prototype.replace` call out to the method `exec` on the RegExp provided, rather than calling the initial value of `RegExp.prototype.exec` or the underlying [[Match]] internal algorithm (or, invoke the _matcher_ internal algorithm defined by the RegExp, in ES2015 terminology).

This proposal suggests reverting 2. and instead using 1. as the sole new mechanism for making RegExps more flexible and subclassable.

## Motivation

The motivation for the proposal is performance, ease of implementation and clean specification. To support the way that `lastIndex` is not updated on certain methods, a trick have to be used. On `RegExp.prototype[Symbol.split]`, the RegExp is actually cloned in order to install the `y` flag, to do a match at just one particular index.

This isn't just an implementation detail. A user who either monkeypatches `RegExp.prototype.exec` or replaces it in a subclass can observe the identity of the regular expression, the `lastIndex` and the flags used. For new proposals, such as the [String.prototype.matchAll](https://github.com/ljharb/String.prototype.matchAll) proposal, it is tempting to use a similar strategy to `split`, of cloning the RegExp in order to isolate the `lastIndex` changes.

However, the extra allocation comes at a real performance cost. It might be possible to fix that performance cost with more advanced implementation techniques, but this has not been demonstrated. The value to the user of the flexibility granted here in ES2015 seems more limited, when a RegExp subclass is free to implement its own `Symbol.split` method already and doesn't need to override `RegExp.prototype.exec` to get there.

It is a bit redundant to have multiple mechanisms for subclasses of `RegExp` to get at their customization. ES2015 gives two, but the `RegExp.prototype.exec` override doesn't work for `String.prototype.split`. It would be rather confusing to have overriding `RegExp.prototype.exec` affect most, but not all, of the RegExp methods. Given that they themselves can be individually overridden, and it is not needed for backwards compatibility with ES5, it seems unnecessary to provide this secondary mechanism.

## What to change in the spec

The method `RegExp.prototype[Symbol.split]` should instead call out to a part of RegExpBuiltinExec which is factored out more like the [[Match]] internal algorithm of ES5. This algorithm returns a tuple giving the results, rather than imperatively mutating the input RegExp. It can be called internally by RegExpBuiltinExec.

The other methods, `RegExp.prototype[Symbol.search]`, `RegExp.prototype[Symbol.match]` and `RegExp.prototype[Symbol.replace]` should call out directly to RegExpBuiltinExec, effectively matching their ES5 behavior and disallowing any replacement for `RegExp.prototype.exec` to take effect in their execution.

## Downside

The major disadvantage is that it is more work to effectively implement a RegExp subclass. Rather than just overriding `RegExp.prototype.exec` and expecting that that code will be reached from all paths, a user will also have to override `RegExp.prototype[Symbol.split]` and three other methods. However, subclassing RegExp is a rather specialized thing to do, and the duplication here may be the least of the users' concerns.

## A less important suggestion

ES2015 also adds some flexibility to the use of flags. Some places where flags are read come from [[OriginalFlags]], crucially including everything that has to with actually executing the matcher. This is necessary so that the RegExp does not need to be recompiled due to dynamically changing flag values. However, other uses of flags are based on invoking the Get internal algorithm on the RegExp object. Users may override this either in subclasses or in monkeypatching the `RegExp.prototype` object (with `Object.defineProperty`).

The new capability leads to some somewhat strange mismatches. For example, the `unicode` flag may be defined in one way as a property and in another way in [[OriginalFlags]]. This will make going between multiple matches Unicode-aware, but matching a particular instance not Unicode-aware. In general, the flexibility allows subclasses to spoof different flags values, but not in a very useful way. The way that `RegExp.prototype[Symbol.split]` invokes the `flags` getter would make it rather difficult to avoid allocating a string just to read the properties. I'd suggest that internal algorithms refer to flags by the [[OriginalFlags]] internal slot, rather than Get invocations.
