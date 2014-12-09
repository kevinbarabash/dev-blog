---
layout: post
title:  "Debugging Javascript with Javascript: Part 3"
date:   2014-11-14 14:16:23
categories: debugging javascript
---

## Review ##
This series of posts describes the development of [stepper], a JavaScript
debugger written in JavaScript.  If you haven't read [part 1] and [part 2] yet,
please do so.  Last time we left off with a step function that can step into,
and out of, and handle return values correctly.

## Overview ##
In this post we'll look at how to deal with functions we can't instrument.  The
code that instruments the source code we want to debug turns any function call
being made into a yield statment that looks like this:

{% highlight javascript %}
{
    gen: foo(),     // foo() returns a generator
    line: 2
}
{% endhighlight %}

The reason we could assume that foo() returns a generator is that foo is a user
defined function and we converted all user defined functions to generators.

If the user wants to make a call to a build-in function like `Math.sqrt()` or
`Console.log()` we can't handle them in the same way because they don't return
generators.  We could create a wrapper function for these function calls.  The
wrapper would return a generator which we would call in the usual fashion and
it would simple call the function with the proper arguments.

This is the approach that Amjad Masad takes with his [debugjs] project.  The
reason why I didn't take this approach is it requires a lot additional code to
get things working correctly.  Not only do all of the arguments have to be
passed through, but so does a pointer to `this`.

Instead, we take a simpler approach.  Call the function and return it's value
in using `yield` like before and have the `step()` function take on the
responsibility of determining whether it's a generator or not.  If it is a
generator, push it on the stack and step in.  If it isn't a generator, treat
result.value.gen as the return value from the function being called and store
it in `retVal` so that we can pass it into the next `next(retVal)` call.

TODO: move to "future work" session... replace this code snippet with some
instrumented code and then an example of how to step through... sleep on it.
<table>
<tr><th>Before:</th><th>After:</th></tr>
<tr><td>
{% highlight javascript %}
point(random(400), random(400));
{% endhighlight %}
</td><td>
{% highlight javascript %}
yield {
    gen: point(
        yield {
            gen: random(400),
            line: 1
        },
        yield {
            gen: random(400),
            line: 1
        }),
    line: 1
}
{% endhighlight %}
</td></tr>
</table>

As each `yield` statement returns it's value back to the stepper, the stepper
must feed that value back into the next call to `.next()`.

Currently, we check if `return.value.gen.toString() === "[object Generator]"`
to determine if what was return is a generator or not.  Looking through the
spec I couldn't find any place where it defines what `.toString()` should return
for generators so at this point it seems that this is convention that Chrome
and Firefox happen to share.

## Future Work ##

How function calls are dealt with needs to be refactored to handle the case
involving nested calls to non-instrumented functions.  Currently function calls
that are on individual lines have yield statements between them, just like normal
statements.  The problem is that with nested function calls in the same expression
there's no place to put the extra yield statements.

Instead we need to remove the extra yield statements after all function calls so
that they all look the same.  The line number information is already encoded in
the yield statement, we just need to a little extra logic to do the right thing
when exit a function call.  Currently we highlight the line of the function
call before advancing to the next statement.  This gives the user some time to
orient themselves before proceeding.  We'd like to maintain this behaviour.  It
shouldn't be difficult give the fact that the call site location is stored in
each frame on the stack.

## stepping into constructors ##

Given `function* foo() { ... }` it is possible to create an object from it by
calling **new**.  If you have yield statements in the constructor you'll have
to call `.next()` until it's done.  The final result's value will be undefined,
unless the constructor returns a value, but this usually isn't the case.
Instead the generator itself will end up being an instance of `foo`.  Here's
an example:

{% highlight javascript %}
function* Point(x,y) {
    yield { line: 1 };
    this.x = x;
    yield { line: 2 };
    this.y = y;
}

var p = new Point(5,10);
p.next();               // => { done: false, value: { line: 1 } }
p.next();               // => { done: false, value: { line: 2 } }
p.next();               // => { done: true, value: undefined }
console.log(p)          // > { x: 5, y: 10 }
{% endhighlight %}

On the surface this appears to work.  Unfortunately, although `p` is an instance
of `Point`, it's also an instance of `GeneratorFunctionPrototype` which means
that it inherits a `next()` method.  Also, any attempts to override this method
are unsuccesful.  Since telling users they can't add a `next` property is out
of the question so we'll need a different approach.




[part 1]:          /dev-blog/{% post_url 2014-11-12-debugging-js-with-js-1 %}
[part 2]:          /dev-blog/{% post_url 2014-11-13-debugging-js-with-js-2 %}
[stepper]:         http://github.com/kevinb7/stepper
[debugjs]:      https://github.com/amasad/debugjs
