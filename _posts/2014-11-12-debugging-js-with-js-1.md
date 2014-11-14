---
layout: post
title:  "Debugging Javascript with Javascript: Part 1"
date:   2014-11-12 14:16:23
categories: debugging javascript
---

## Background ##

This journey started when I was invited to participate in the Khan Academy
hackweek (a few weeks ago).  I had been playing around with the live-editor
(used in the KA CS curriculum), creating issues, committing bug fixes and
minor improvements.  I knew I wanted to do something big.

For those who haven't used the live-editor, it allows users to write programs
using JavaScript and the processing.js framework.  The cool thing about it is
that it live updates the output canvas.  It even has these cool little sliders
that you can drag to update variables.

One thing that I really missed was having a debugger.

The live-editor does a really good job of sandboxing code and simplifying the
coding experience by providing aids like the "Oh noes!" that appears to alert
users when (and where) they've made a mistake.

## Goals ##

Debuggers can be complicated especially for novices.  That's why I decided to
create two debuggers, one for beginners and one for advanced users.  The
beginner debug wouldn't have any breakpoints and would only have the following
controls:

- begin – reset the program and move the program counter to the first line
- step – step in
- end – run to the end of the program

The advanced debugger would allow for breakpoints (in fact it requires at least
one breakpoint ot be set) and have a more classic set of debugger controls:

- restart – reset the program and start running from the start until the first
breakpoin is hit
- step in
- step over
- step out
- continue – runs until the next breakpoint is hit

## Existing Solutions ##

Before starting this project, I knew that debugging JavaScript with JavaScript
is possible.  There is a project called [debugjs] by Amjad Masad that uses ES6's
generators to enable step-by-step debugging.  My original plan was to integrate
this project into the live-editor and call it day.  Unfortunately, there were
some serious issues that caused me to start writing my own debugger.  These
issues included:

- It didn't work properly with processing.js, even in an isolated test app.
For some reason setting the fill/stroke color didn't work properly.
- debugjs seemed more complicated that it needed to be (at least for the purposes
of the hack).
- debugjs doesn't step into constructors.

## Generators ##

Some background info on generators before we get started.  Generators are part
of the upcoming ES6 standard and provide a way to re-enter a function and
continue execution from specific points.  These points are defined using a new
keyword, `yield`.  The spec also defined a new syntax to defined generator
functions using `function*`.  Here's an example of a function that returns a
generator.

{% highlight javascript %}
function *incBy(x) {
  var i = 0;
  while (i < 10) {
    yield i;
    i += x;
  }
}
{% endhighlight %}

The function `incBy(x)` is not a generator itself.  Instead it returns an object
that has a method `next()` which returns an object containing two properties:

- `value` – the value that passed to the last `yield` statement
- `done` – `true` if the function comes to the end or hits a `return` statement

Here's how the `incBy(x)` would be used:

{% highlight javascript %}
var incBy3 = incBy(3);

console.log(incBy3.next());  // => { value: 0, done: false }
console.log(incBy3.next());  // => { value: 3, done: false }
console.log(incBy3.next());  // => { value: 6, done: false }
console.log(incBy3.next());  // => { value: 9, done: false }
console.log(incBy3.next());  // => { value: undefined, done: false }
{% endhighlight %}

## Basic Technique ##

In order to debug code, I wanted to be able to step over each line, one by one,
this meant that I need to create a debug version of user code that had `yield`
statements inserted between each line.  This would allow me to execute the user
code one step at a time.  Here's my first attempt:

<table>
<tr>
<th>Source Code</th>
<th>Debug Code</th>
</tr>
<tr>
<td>
{% highlight javascript %}
function foo() {
  console.log("hello, ");
  console.log("world!");
}
{% endhighlight %}
</td>
<td>
{% highlight javascript %}
function foo*() {
  yield;
  console.log("hello, ");
  yield;
  console.log("world!");
}
{% endhighlight %}
</td>
</tr>
</table>

To "step" through the debug code one would do the following:

{% highlight javascript %}
var gen = foo();

gen.next();
gen.next();     // hello,
gen.next();     // world!
{% endhighlight %}

## Transforming Code ##
In order to insert these `yield` statements in a robust way I turned to [esprima],
a well known JavaScript parser.  It produces an AST conforming to the [Parser API]
defined by Mozilla's SpiderMonkey JavaScript runtime.

Getting an AST is the easy part.  To insert yield statements we need to traverse
the AST, and after each appropriate statement insert a yield statement.  At the
time I wasn't aware the estraverse project and ended up writing my own called
ast-walker.  I will be replacing ast-walker with estraverse in the future.



## Line Numbers ##

Being able to step through a program is a good start but we also need to update
the UI by highlighting the current line.  Since we don't necessarily know the
line number for the first line of a function we need to start each function with
a yield statement that provides that information.  Also, each subsequent yield
will return the line number for the next line to be executed.  This will allow
us to highlight that line in the editor before actually executing it.

The line numbers themselves can be retrieved from the AST.  One of the options
esprima has is to include location information in each node of the AST.  This
includes both the start and the end.  For this debugger I used `loc.start.line`
for the line number.

<table>
<tr>
<th>Source Code</th>
<th>Debug Code</th>
</tr>
<tr>
<td>
{% highlight javascript %}
function foo() {
  console.log("hello, ");
  console.log("world!");
}
{% endhighlight %}
</td>
<td>
{% highlight javascript %}
function foo*() {
  yield { line: 1 };
  console.log("hello, ");
  yield { line: 2 };
  console.log("world!");
}
{% endhighlight %}
</td>
</tr>
</table>

Now that we're getting line numbers back from the generator it's a good time
to create a `step()` function.  Here's what stepping through the code looks like:

{% highlight javascript %}
var gen = foo();

function step() {
    var result = gen.next();
    editor.highlightLine(result.value.line);
    return result;
}

step()
step();     // hello,
step();     // world!
{% endhighlight %}




[debugjs]:      https://github.com/amasad/debugjs
[esprima]:      http://esprima.org/
[Parser API]:   https://developer.mozilla.org/en-US/docs/Mozilla/Projects/SpiderMonkey/Parser_API
