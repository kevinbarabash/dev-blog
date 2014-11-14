---
layout: post
title:  "Debugging Javascript with Javascript: Part 2"
date:   2014-11-13 14:16:23
categories: debugging javascript
---

## Review ##
This series of posts describes the development of [stepper], a JavaScript
debugger written in JavaScript.  If you haven't read [part 1] yet, please do so.
Last time we left of with a basic `step()` function which runs a function one
step at a time and updates the line number in the editor.

## Programs ##

If you're debugging code, chances are you'll have some code that's not inside
of a function.  Maybe something like this:

{% highlight javascript %}
var x = 5;
console.log("x = " + x);
{% endhighlight %}

If this is our program and we want to be able to step through we first need to
wrap it in a function which returns a generator.

<table>
<tr><th>Before</th><th>After</th></tr>
<tr>
<td>
{% highlight javascript %}
var x = 5;
console.log("x = " + x);
{% endhighlight %}
</td>
<td>
{% highlight javascript %}
function* __program__() {
    yield { line: 1 };
    var x = 5;
    yield { line: 2 };
    console.log("x = " + x);
}
{% endhighlight %}
</td></tr>
</table>

The tools used to perform this transform are __esprima__ and __escodegen__.
__esprima__ is a parser which produces an AST for a string and __escodegen__ is
a code generator produces a string containing code from an AST.  Here's what the
basic flow looks like:

{% highlight javascript %}
var ast = esprima.parse(sourceCode);

// modify the ast

var debugCode = escodegen.generate(ast);
{% endhighlight %}

I've left out the modify step because I don't think it adds a lot to this
description of how to debug JavaScript with JavaScript.  If you're interested
please look at the code.  In particular you'll want to pay close attention
to the `_createDebugGenerator(code)` method in stepper.js.  If there's lots of
interest I can write a separate post talking just about all these AST transforms
were accomplished.

## Context ##
The live-editor uses the processing.js.  It uses the `with` statement to inject
properties from an `processing` object into the current scope.  This allows
users to write `rect(100,100,50,50)` instead of `processing.rect(100,100,50,50)`.
We'd like to the same with our debugger so that it can be used in the CS
curriculum at KA.  In order to do this we need our function to look like this:

{% highlight javascript %}
function* __program__() {
    with (processing) {
        yield { line: 1 };
        var x = 5;
        yield { line: 2 };
        console.log("x = " + x);
    }
}
{% endhighlight %}

The problem with this approach is that it's inflexible.  If someone changes the
name of `processing` to `Processing` we have to change the code.  Also, if other
people want to use they'd have to modify the code.  In order support different
contexts easily we'd like to be able to pass a `context` param to the function
that creates our debug generator.

Originally I was planning to support multiple contexts using a more complicated
scheme, but this required multiple `with` statements which reduces performance.
See: [multiple with statements].

In order to pass an argument we need to give it a param, but we also need to have
a reference to it before we can call it.  Also, it would be nice if we didn't
have this `__program__` identifier messing up our scope.  Here's the solution
used by stepper.js.

{% highlight javascript %}
var debugCode = 'return function* (context) {
    with (context) {
        yield { line: 1 };
        var x = 5;
        yield { line: 2 };
        console.log("x = " + x);
    }
}';

var debugFunction = new Function(debugCode);
var debugGenerator = debugFunction(context);
{% endhighlight %}

## Stepping Into Functions ##

Now that we've wrapped our program with a function that produces a generator and
we can step over instruction, how do we go about stepping into user defined
functions within the program.  I reason why I specified "user defined" functions
is that in order to step through a function it needs to be a generator which
precludes any code that hasn't been transformed by us.

Since we know how to step through functions which return generators, all we need
to do is transform all functions in the program into generators.  Whent the
function is called it will return a generator which we will `yield` back to the
debugger.  Here's an example of what this looks like:

<table>
<tr><th>Before</th><th>After</th></tr>
<tr><td>
{% highlight javascript %}
function foo() {
    console.log('foo');
}
foo();
{% endhighlight %}
</td><td>
{% highlight javascript %}
yield { line: 1 };
function* foo() {
    yield { line: 2 };
    console.log('foo');
}
yield { line: 4 };
yield { gen: foo(), line: 4 };
{% endhighlight %}
</td></tr>
</table>
__Note__: I've omitted the wrapper described in the previous section.  It's still
needed.

When `foo()` is called, it returns a generate and `yield` returns it back to the
debugger.  Because we now have multiple generators to worry about (the program
generator and the generator return by `foo`) we need to introduce a stack to
keep track of which generator we're stepping through.

{% highlight javascript %}
var stack = new Stack();
stack.push(debugGenerator); // push the base generator (program)

function step() {
    var currentGen = stack.peek();
    var result = currentGen.step();
    if (result.done) {
        // step out
        stack.pop();
        editor.highlightLine(result.value.line);
    } else if (result.value.gen) {
        // step in
        stack.push(result.value.gen);
        editor.highlightLine(result.value.line);
    } else {
        // step over
        editor.highlightLine(result.value.line);
    }
    if (stack.isEmpty()) {
        return DONE;
    }
}

function run() {
    while (step() !== DONE);
}
{% endhighlight %}

While this new `step` function will step through every function call, it gets
the line numbers wrong.  In order to correct this we need to add some line
number information to the stack frames.  We also need to take an additional
step when stepping into functions.  All functions have a `yield` right before
any other code which provides the correct line number.  Because of this we don't
have to worry about whether the first value that yielded contains a generator,
we know it doesn't.  Here's the corrected code:

{% highlight javascript %}
var stack = new Stack();
stack.push({
    gen: debugGenerator,
    line: 0
});

function step() {
    var currentFrame = stack.peek();
    var result = currentGen.step();
    if (result.done) {
        // step out
        var finishedFrame = stack.pop();
        // the "line" for each frame is the call site location
        editor.highlightLine(finishedFrame.line);
    } else if (result.value.gen) {
        // step in
        stack.push(result.value);
        // take another step to get the correct line number
        result = currentFrame.gen.step();
        editor.highlightLine(result.value.line);
    } else {
        // step over
        editor.highlightLine(result.value.line);
    }
    if (stack.isEmpty()) {
        return DONE;
    }
}
{% endhighlight %}

## Return Values ##

The current approach works fine for functions that don't return values.  If you
try to step through a program that returns values, you'll be able to step, but
the return values don't end up where they're supposed to go.  Before we look at
how to fix the problem, let's look first at how return values work with genarators.

{% highlight javascript %}
function* foo() {
  yield 1;
  yield 2;
  return 3;
}

var gen = foo();
gen.next();         // => { done: false, value: 1 }
gen.next();         // => { done: false, value: 2 }
gen.next();         // => { done: true, value: 3 }
{% endhighlight %}

As long as the function returns a value, when the generator is done, you can
still get at the return value which is stored in the `value` property.

This is fine, but how do you deal with code where the return value is used
right away in an expression like the following:

{% highlight javascript %}
function add(x, y) {
    return x + y;
}
console.log("1 + 2 = " + add(1,2));
{% endhighlight %}

It turns out generators allow for bidirectional "communication" between the
generator and the code calling `next()`.  Up till now we've been using `next()`
and `yield` to get values out of the generator, namely the line number and
potentially another generator.  It turns out that you can also pass data back
into generator by pass an argument to `next()`.  When a value is passed to `next()`
it replaces the last `yield` statement that was executed in the generator.

This is super important so I'll state it again:  Passing a value to `next()`
replaces the last `yield` statement that was executed in the generator.

Here's an example:

{% highlight javascript %}
function* foo() {
    console.log(yield 5);
}
gen = foo();
foo.next();     // => { done: false, value: 5 }
foo.next(20);   // => { done: true, value: undefined }
                // > 20
{% endhighlight %}

The `console.log` inside of `foo` ends up printing 20 because `yield 5` was
replaced by 20.  Armed with this knowledge we can update our `step` function
to take this into account.

{% highlight javascript %}
var stack = new Stack();
stack.push({
    gen: debugGenerator,
    line: 0
});

// only set this when we return from a function call
var retVal = undefined;

function step() {
    var currentFrame = stack.peek();
    var result = currentGen.step(retVal);
    if (result.done) {
        retVal = result.value;
        // step out
        var finishedFrame = stack.pop();
        // the "line" for each frame is the call site location
        editor.highlightLine(finishedFrame.line);
    } else if (result.value.gen) {
        // step in
        // reset retVal because we're not exiting a function
        retVal = undefined;
        stack.push(result.value);
        // take another step to get the correct line number
        result = currentFrame.gen.step();
        editor.highlightLine(result.value.line);
    } else {
        // step over
        // reset retVal because we're not exiting a function
        retVal = undefined;
        editor.highlightLine(result.value.line);
    }
    if (stack.isEmpty()) {
        return DONE;
    }
}
{% endhighlight %}

[part 1]:                       /dev-blog/{% post_url 2014-11-12-debugging-js-with-js-1 %}
[stepper]:                      http://github.com/kevinb7/stepper
[multiple with statements]:     http://jsperf.com/multiple-withs