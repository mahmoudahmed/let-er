# let-er

ES6 is bringing the `let x = "foo"` syntax to JavaScript, which basically hijacks any block and scopes your declaration to that block (instead of hoisting to the containing function as in ES5 and below).

**Two problems, however:**

1. ES6 is a long time from being fully ubiquitous.
2. And even when ES6 arrives, the more preferable `let ( .. ) { .. }` block syntax was rejected and so won't be valid. This is most unfortunate, since that syntax has a more readable form, creating explicit blocks for block-scoping instead of hijacking them.

*let-er* solves both these problems. By simply writing `let ( .. ) { .. }` style code, and then running your code through *let-er* as a build-step, your block-scoping will work exactly as you expect, **today**.

By default, *let-er* transpiles your `let ( .. ) { .. }` blocks into standard ES6 `{ let .. ; .. }` blocks. If you pass the [`es3` option setting](#options), you can even target ES3+ browsers, using the `try..catch` hack.

## What does it look like?

Write block-scoped code like this:

```js
let (x = "foo") {
	console.log(x); // "foo"
}

console.log(x); // Reference Error!
```

**NOTE:** *let-er* does **not** touch `let ..;` declaration syntax (similar to `var ..;`), only the (more preferable and more readable) `let ( .. ) { .. }` statement syntax. If you use `let ..;` declarations, *let-er* will simply skip over them.

## Why block-scoping anyway?

You may be wondering what's the big deal with block-scoping? What difference does it make?

In my ["For and against `let`"](http://davidwalsh.name/for-and-against-let) blog post, I lay out the case in detail. But let's briefly break the question down into two parts:

1. How does block-scoping affect my [coding style/maintainability/behavior](#coding-stylemaintainabilitybehavior)?
2. How does block-scoping affect my [code performance](#code-performance) (memory, GC, etc)?

### Coding Style/Maintainability/Behavior

There's a concept in computer science called the "Principle of Least Exposure", which is broadly applicable in a lot of areas, but in particular here, we are using that to mean: "expose a variable and its value for only as long as it's needed, no more, no less."

Take this code snippet:

```js
function foo() {
	var i, ret = [];

	// 100+ LoC doing other stuff with `ret` only

	for (i=0; i<ret.length; i++) {
		// ... other stuff with `i` and `ret`
	}
}
```

The fundamental question is, is it a good thing to have defined `i` as a "global" (to the scope of function `foo()` of course) variable, or would it have been better to keep it around (aka, "expose it") only for the loop that uses `i`?

What if we had done:

```js
function foo() {
	var ret = [];

	// 100+ LoC doing other stuff with `ret` only

	for (var i=0; i<ret.length; i++) {
		// ... other stuff with `i` and `ret`
	}
}
```

That keeps `i` only referenced in the place he's used. That's an improvement.

And yet, because of declaration hoisting, that snippet will behave exactly the same as the previous snippet. The engine sees no difference.

Another example:

```js
function foo() {
	var ret = [];

	// 100+ LoC doing other stuff with `ret` only

	if (ret.length > 10) {
		var i = ret.length;
		// do other stuff with `i` now, but only inside this `if`
	}
}
```

See how we've made it clear from a coding style perspective that `i` is only for that part of the function?

By contrast, `ret` is a variable we use throughout the function `foo()`, so it's perfectly fine and suitable to function-scope it.

Unfortunately, we've created a little bit of cognitive dissonance here between the code style of where `i` is declared, and what the actual scoping of `i` is, because `i` will be hoisted to the whole function regardless of our code style.

That means if you accidentally try to reference a variable `i` elsewhere in the code, maybe as a typo, you'd wish that you got some sort of helpful error like a "ReferenceError" that points out your mistake, but you won't get that, because `i` in fact exists everywhere inside of `foo()` even if it has no meaningful value yet.

Herein we see a code maintainability (aka "style") benefit of the new ES6 `let` keyword for block-scoping. By doing this:

```js
function foo() {
	var ret = [];

	// 100+ LoC doing other stuff with `ret` only

	if (ret.length > 10) {
		let i = ret.length;		// <===== Notice the `let` !!?!?!!?!
		// do other stuff with `i` now, but only inside this `if`
	}
}
```

Now, the behavior of `i` will be to be block-scoped to that part of the code, and it will be clear not only stylistically but also functionally that `i` doesn't exist anywhere but in that block. You'll get helpful "ReferenceError" errors if you try to reference `i` elsewhere. Yay.

### Code Performance

Now, let's examine how `let` ***may*** affect your code's performance. I say "may" for a couple of reasons. First, the number of engine implementations of `let` is pretty limited, so we can't *really* test in the real world just how much it will or won't affect performance. We can make intelligent guesses about the possibilities.

Secondly, what the engine can do in theory and what it *will* do in the real-world are often very different. These are implementation details and we should be careful not to get too much "into the weeds" here. Only those who actually make the engines are qualified to *really* obsess too much here.

But let's reason a little bit.

Consider:

```js
function foo() {
	// 1000+ LoC of a bunch of complex stuff
}

function bar() {
	// a small simple function
	// note: does not use `foo()` at all
}

setTimeout(bar,60000); // make bar's closure stay around awhile

for (var i=0; i<10; i++) {
	foo(); // only places `foo()` is used
}
```

The `bar()` function has a closure over everything we see here in this scope, including the `foo()` function it doesn't use at all. Turns out `foo()` is a big hairy complex function so its memory footprint is not "zero".

For the 60 seconds that `bar()` is kept alive, his closure over this whole scope is also kept alive, meaning `foo()` is kept alive. But does `foo()` really need to be kept alive? Depends.

If we (and by "we", I mean the engine, of course!) know for sure that `bar()` never needs `foo()`, we might be able to make an intelligent implementation optimization that culls `foo()` out of the closure that `bar()` keeps around. We might, we might not. If `bar()` has anything inside him that we can't lexically analyze, like a `with` or `eval`, all bets are off the table. Who knows if `bar()` needs `foo()` or not? Better safe than sorry. Also, who knows if the engine actually does smart closures or not?

In theory they might. But in practice, the safer bet is to assume they don't or can't, and code a little defensively.

Here's where `let` block-scoping can help. It acts as a very clear signal to the engine what we intend for the lifetime of `foo()`. Therefore, the engine doesn't have to guess nearly as much. It can tell for sure. Consider:

```js
function bar() {
	// a small simple function
	// note: does not use `foo()` at all
}

setTimeout(bar,60000); // make bar's closure stay around awhile

{
	let foo = function foo() {
		// 1000+ LoC of a bunch of complex stuff
	};

	for (var i=0; i<10; i++) {
		foo(); // only places `foo()` is used
	}
}
```

Simple change, but *potentially* a big performance difference. Now, it's clear that `foo()` only exists inside that block. `bar()` wouldn't have access to it anyway, even if it tried, so `foo()` definitely doesn't need to be kept around while `bar()` is waiting to execute. The engine has a much clearer signal that it's OK to free up `foo()` from the closure, to garbage collect it.

Will the engines do this? I dunno. You dunno, probably. But they *can*. And they *might*. And they *might* be able to more efficiently than if you hadn't used block-scoping.

## How does *let-er* work?

*let-er* will, by default, transform your `let ( .. ) { .. }` style blocks into standard ES6-compatible code.

This:
```js
let (x = "foo") {
	console.log(x); // "foo"
}

console.log(x); // Reference Error!
```

Becomes this:
```js
{ let x = "foo";
  console.log(x); // "foo"
}

console.log(x); // Reference Error!
```

Of course, once you have standard ES6 code, you can either run it directly in an ES6 compatible browser/environment, or you can then run the whole code base through a ES6-ES3/5 transpiler, so that you can target older environments.

In otherwords, *let-er* should be used as a preprocessor to bring your code to ES6 compliance, and then go from there as you normally would.

If you don't plan to use another ES6 transpiler, you can make *let-er* target ES3+ compatibility, such that the code produced would be:

```js
try{throw "foo"}catch
/*let*/ (x /* = "foo" */) {
  console.log(x); // "foo"
}

console.log(x); // Reference Error!
```

A bit hacky? Sure. But it works. And you get to write nice, maintable code with the benefits of block-scoping. Who cares what the transpiled code looks like, right? :)

**NOTE:** By default, shown here are the extra comment annotations to make the transpiled code a bit more readable. You can suppress these default comment annotations with the `annotate` [option setting](#options), in which case the ES3+ code produced would be:

```js
try{throw "foo"}catch(x){
  console.log(x); // "foo"
}

console.log(x); // Reference Error!
```

### Multiple `let`-declarations

Multiple `let`-declarations are supported:

```js
let (x = "foo", y = "bar") {
  console.log(x, y); // "foo" "bar"
}

console.log(x, y); // Reference Error!
```

The ES6 transpiled code will look like this:

```js
{ let x = "foo", y = "bar";
  console.log(x, y); // "foo" "bar"
}

console.log(x, y); // Reference Error!
```

The ES3+ transpiled code will look like this:

```js
try{throw "foo"}catch
/*let*/ (x /* = "foo" */) {
try{throw "bar"}catch
/*let*/ (y /* = "bar" */) {
  console.log(x, y); // "foo" "bar"
}}

console.log(x, y); // Reference Error!
```

Or, ES3+ without annotations:

```js
try{throw "foo"}catch(x){try{throw "bar"}catch(y){
  console.log(x, y); // "foo" "bar"
}}

console.log(x, y); // Reference Error!
```

## Performance

Of course, the ES6 transpilation has zero performance difference to regular ES6.

But, you may be wondering if there's some crazy performance hit to the ES3 `try..catch` hack. So, here's several thoughts to address that concern:

1. Comparing block-scoped code to non-block-scoped code performance-wise wouldn't really tell you what you might expect. You might intuitively be wondering if the performance hit is small enough (or zero!) so that you can use block-scoping "for free". It **certainly is not**.

   Either you need/want block-scoping, or you don't. But asking if using block-scoping is poorer performance than not using block-scoping is not a terribly useful question. The merits of the block-scoping functionality are the important question.

2. A fairer comparison, though still skewed, is comparing different approaches for getting "block scoping". The main alternative to the `try..catch` hack which *let-er* uses is the IIFE (creating an inline, auto-executing function to get an extra function scope block where you want block-scoping). So, you *can* examine [how `try..catch` compares to IIFE](http://jsperf.com/block-scope-iife-vs-catch).

   As you'll see there, `try..catch` is about 10-20% slower on average than an IIFE. This may seem like the nail in the coffin for this approach. However, there's ***a major caveat to using an IIFE***. It has some significant side-effects.

   Namely, the value of `this`, the meaning of `return` `break` `continue`, and other such things, are very different inside an IIFE, whereas these things are not affected by wrapping a `catch(..) { .. }` block around any arbitrary code.

   So, even the IIFE vs. `try..catch` isn't a totally fair comparison, but it's close enough to be relevant. It does show that the `try..catch` hack isn't orders of magnitude worse or anything crazy like that. It's a bit worse than an IIFE, but in exchange you get truer block-scoping that is less intrusive/destructive to the code.

3. This transpiling to ES3 via `try..catch` is an admitted polyfill. Polyfills almost always have worse performance than their newer native counterparts. ES6 is giving us the `let` keyword for real block-scoping, and it certainly will be faster than the polyfill hack.

   Fortunately, *let-er* defaults to ES6 code if you happen to only care about ES6 environments (node.js for instance). As you can see above, the ES6-only targeting transpiles to usage of the native `let ..;` declaration syntax, which should have all the best native performance you can get.

## API

The `compile(..)` API method takes code and detects if there are any matching `let ( .. ) { .. }` style blocks that it needs to transpile. You get a single code string back, ready to go.

```js
letEr.compile('let(x="foo"){console.log(x);}');
```

### Additional API methods

The API also includes `lex(..)`, `parse(..)`, and `generate(..)`:

* `lex(..)` takes code and returns an array of tokens from the lexing of the code.
* `parse(..)` takes the array of tokens and parses it for let-blocks, and returns an AST (tree: an array of nodes and nested nodes).
* `generate(..)` takes the AST formatted as it comes from `parse(..)` and produces the code string, the same as it comes from the `compile(..)` API method.

If you inspect the token stream or the AST, each element has a `type` attribute with a numeric value. The *let-er* API provides constants for these values to make it easier to interpret the element types.

For example, `letEr.TOKEN.LET_HEADER` is the value that identifies the token list (nested tokens) that represents the let-block's declaration header. `letEr.NODE.LET_BLOCK` by contrast is the value for a node in the AST that represents a full let-block, including the nested declarations and the content inside the `{ .. }` block.

*let-er* uses [literalizer](https://github.com/getify/literalizer), which means that the token stream and the AST will have each of *literalizer*'s identified literals as separate elements. Those elements all have a `literal` property with a value that corresponds to the [constants on the literalizer API](https://github.com/getify/literalizer#complex-literals), such as `LIT.SEGMENT.STRING_LITERAL` for, obviously, string literal elements.

Most people will not need to use these additional *let-er* API methods, but if you do need to perform extra analysis or transforms, the API provides you that flexibility. **NOTE:** Be careful to not modify/invalidate the format of these element structures, or *let-er* will likely not be able to consume them again for the next step.

## CLI

*let-er* comes with a node.js tool called `leter` (note the lack of `-`), in the "bin/" directory, which is a CLI tool for compiling your let-block JS code.

Here are the options for the CLI tool:

```
usage: leter opt [, ..opt]

options:
--help                 show this help

--ignore-warnings      ignore and suppress any warnings
--no-annotate          do not annotate ES3 output
--es6                  target ES6 for output (use native let keyword)

--compile              compile the JS stream from 'stdin'
--compile=file         compile a JS file
```

If the `--compile` flag is passed, it will read code in from *stdin*.

The `--compile=file` flag can (also) be passed multiple times, once for each file you want to compile.

```
bin/leter --compile=some/file1.js --compile=some/file2.js
echo ".. some code .." | bin/let --compile --compile=some/file3.js
```

## Options

There are two options you can set that control the type of code produced by *let-er* (aka, the `generate(..)` step).

* `letEr.opts.es3` (boolean; default: `false`) - If set to `true`, will target ES3+ environments with the `try{throw "foo"}catch(x){ .. }` hack syntax.

* `letEr.opts.annotate` (boolean; default: `true`) - If set to `true`, will output additional code comments (as shown in the above example snippets) to annotate to make the ES3+ compiled code more readable/trackable to the original code. Otherwise, only the bare minimum code will be output.

   **NOTE:** This option only matters if the `es3` option is set, and will have no effect otherwise.

## Warnings

If there are any warnings/notices produced as part of the lexing or `let` processing, they will be populated in the `letEr.warnings` array. You can call `letEr.reset()` to clear out the warnings list in between separate compilations.

## License

The code and all the documentation are released under the MIT license.

http://getify.mit-license.org/
