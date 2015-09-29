# Partial revert of ES2015 RegExp semantics

*ES2015 makes regular expressions subclassable.* There are a couple things it changes vs ES5 which allow behavior to be overridden:
1. String functions which call things related to RegExps can be overridden using Symbol-named methods on RegExps, e.g., `RegExp.prototype[Symbol.match]`, `RegExp.prototype[Symbol.search]`, etc.
2. `String.prototype.split` and `String.prototype.search` call `RegExp.prototype.exec`, rather than calling the underlying [[Match]] internal algorithm (or, invoke the _matcher_ internal algorithm defined by the RegExp, in ES2015 terminology). Previously, only `String.prototype.match` and `String.prototype.replace` invoked `RegExp.prototype.exec`.

This proposal suggests reverting 2. and instead using 1. as the sole new mechanism for making RegExps more flexible and subclassable. To reinforce the lack of flexibility with flags in 3., the [[OriginalFlags]] mechanism would be kept, but all internal algorithms which look up flag values would read them from [[OriginalFlags]] directly, rather than using Get().

The motivation for the proposal is performance, ease of implementation and clean specification. To support the way that `lastIndex` is not updated on certain methods, some tricks have to be used:
- On `RegExp.prototype[Symbol.split]`, the RegExp is actually cloned in order to install the `y` flag, to do a match at just one particular index.
- On `RegExp.prototype[Symbol.search]`, which is used to implement `String.prototype.search`, the `lastIndex` is saved and restored across the call to the RegExpExec internal algorithm.

In both of these cases, it isn't just an implementation detail. A user who either monkeypatches `RegExp.prototype.exec` or replaces it in a subclass can observe the identity of the regular expression, the `lastIndex` and the flags used. For new proposals, such as the [String.prototype.matchAll](https://github.com/ljharb/String.prototype.matchAll) proposal, it is tempting to use a similar strategy to `split`, of cloning the RegExp in order to isolate the `lastIndex` changes. However, this comes at a real performance cost.
