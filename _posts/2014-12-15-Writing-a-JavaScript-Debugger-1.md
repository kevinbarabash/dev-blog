---
layout: post
title:  "Writing a JavaScript Debugger: Part 1"
date:   2014-12-15 14:16:23
categories: javascript debugging
---

This is part one in a three part series detailing the implementation of a 
JavaScript debugger for using with Khan Academy's [live-editor].

## Goals ##

Why write a JavaScript debugger when there's a perfectly good one included in
your browser?  The problem with debuggers that are built into the browser is 
that they're not integrated with the code editors used in on-line programming
courses such as the ones offered by Khan Academy, Code Academy, etc.

The main goal of writing a debugger in JavaScript was to be able to integrate
it with Khan Academy's live-editor so that students could step through their 
programs.  A secondary goal was to be able to provide user interfaces of varying
complexity so that students could ease their way into debugging.  

They could start out with a simple interface that starts the program paused and 
then either step(in) or jump to the end.  As they gain experience they would
graduate to more complex interfaces that allow for setting/clearing breakpoints, 
jumping in/out/over, and eventually viewing call stacks and inspecting local 
variables.

## Baby Steps ##

I start this project as part of the Khan Academy Hack Week at the end of October.
At the time I thought I'd just use Amjad Masad's excellent project [debugjs]. 
Unfortunately, there were some serious issues with it.  I wasn't able to get it
to work with [processing-js] which is used in live-editor.  It would draw shapes,
but it wouldn't set colors correctly.  

Debugjs uses an iframe to isolate the execution environment of the program being 
debugged.  I had a hunch that the iframe was causing the issue.  I thought about
trying to modify debugjs to remove the iframe requirement, but since I only had
a week I thought it'd be quicker to write a simple debugger that could only step
in/out/over.  That's where the project name [stepper] came from.

Stepper uses a similar approach to debugjs, converting all functions to ES6
Generators and inserting `yield` statements in between each line so that each
time the generator's `next()` method is called another line will be executed.
For a good introduction on how generators work please have a look at
[The Basics Of ES6 Generators] by David Walsh.

The main difference is that debugjs wraps up each function call in something 
called a `thunk` which encapsulates a generator wraped function call, `this`,
and the function's arguments.  More details on this approach can be found in
this [blog post] by Amjad Masad.

__debugjs__:

    foo();  // becomes the following:
    
    yield __thunk(function *thunk() {
        return foo();
    }, this, arguments);


Stepper simplifies function calls by yielding a the result of a function call
without the thunk.  

__stepper__:

    foo();  // becomes the following:
    
    yield foo();

## Stepper Details ##

Actually, this is a simplification in both cases because the `yield` statements
can also return the current line number or even a list of in scope variables.
Here's a more complicated program:

    var x = 5;
    var y = 10;
    var z = add(x, y);
    
Which becomes (with enclosing generator):

    function *() {
        var __scope__ = {
            x: undefined,
            y: undefined
        };
        with (__scope__) {
            yield {
                line: 1
                scope: __scope__
            },
            var x = 5;
            yield {
                line: 2
            }
            var y = 5;
            yield {
                line 3
            }
            yield {
                gen: add(x, y)
            }
        }
    }

Because not all functions get converted to generators the code for stepping 
through a program requires some extra logic to either step through a function 
that's been converted or just call a function that hasn't been.  Only functions 
that are defined as part of the code being debugged are converted to generators.

    stepIn() {
        var result;
        if (result = this._step()) {
            if (result.value && result.value.hasOwnProperty('gen')) {
                if (_isGenerator(result.value.gen)) {
                    this.stack.push({
                        gen: result.value.gen,
                        line: this.line
                    });
                    this.stepIn();
                    return "stepIn";
                } else {
                    this._retVal = result.value.gen;
                    if (result.value.stepAgain) {
                        result = this._step();
                    }
                }
            }
            if (result.done) {
                this._popAndStoreReturnValue(result.value);
                return "stepOut";
            }
            return "stepOver";
        }
    }

The `_step()` function calls `next()` on the generator at the top of the call
stack and takes care of housekeeping operations such as keeping track of the 
current value returned by the generator.  Each time `next()` is called on a 
generator object it can return a value.  When dealing with nested function calls
such as `sqrt(add(mul(3,3), mul(4,4)))`, the value returned by `next()` must be
passed into the next call to `next()`.  Here's how `_step()` takes care of that:

    private _step() {
        if (this.stack.isEmpty) {
            return;
        }
        var frame = this.stack.peek();
        var result = frame.gen.next(this._retVal);  // pass the previous value
        this._retVal = undefined;                   // clear the value

        // if the result.value contains scope information add it to the
        // current stack frame
        if (result.value) {
            if (result.value.scope) {
                this.stack.peek().scope = result.value.scope;
            }
            if (result.value.name) {
                this.stack.peek().name = result.value.name;
            }
            if (result.value.line) {
                frame.line = result.value.line;
            }
        }
        return result;
    }

The value of `this._retVal` is only when a function call completes because this
is the only time when a value needs to be passed to `next`.  I should highlight
that the `gen` property of the `yield` statements here is being overloaded to
be either a generator object or the result of a regular function call.  This 
unfortunately can't be avoid because it's impossible to tell at compile if some
expressions are calling functions which (or may not) have been converted to 
generators.

Because stepper uses `yield` statements to both return generators to be executed
and line numbers, sometimes we need to take an extra step to get the next line
number.  If there's multiple function calls on a line only the last function call
will have the `stepAgain` property.

## Local Variables ##

    function *() {
        var __scope__ = {
            x: undefined,
            y: undefined
        };
        with (__scope__) {
            ...
        }
    }

Although the function body is wrapped in a `with (__scope__) { ... }` statement,
the only variables defined within `__scope__` are locally defined variables.
These are determined by the excellent [escope] library.  

The `with` statement is quite powerful because any locally defined variables 
will get subsumed by the property in `__scope__` with the same name.  In the 
case of function parameters, they are assigned the value of the parameter name.

The `arguments` array is not included in `__scope__` because this debugger will
intially be used by novice developers.  In the future, this will be configurable
so that we can support advanced developers as well.

The `__scope__` variables is returned as part of the first `yield` statement.  It
is exposed to developers via the `currentScope` property on a `Debugger` instance.

## Objects ##

Object instantiation is interesting because while you call a generator function
using the `new` keyword, most constructors don't return a value and thus the 
generator's result's value is undefined.  In order to deal with this situation,
I borrowed debugjs' approach which is to create an `__instantiate__` function
and replace all `new ClassFn()` calls with `__instantiate__(ClassFn, ClassName)`.
Here's the implementation:

    __instantiate__ = (classFn, className) => {
        var obj = Object.create(classFn.prototype);
        var args = Array.prototype.slice.call(arguments, 2);
        var gen = classFn.apply(obj, args);

        this.onNewObject(classFn, className, obj, args);

        if (gen) {
            gen.obj = obj;
            return gen;
        } else {
            return obj;
        }
    }
    
__Note:__ Stepper uses some of the new ES6 features such as arrow functions via 
TypeScript.

`this.onNewObject` is a callback that can be defined and is used by 
`ProcessingDebugger` to notify live-editor when a new object has been created so
that it can keep track of live instances.  `ProcessingDebugger` is a subclass
of `Debugger` which adds support for debugging programs using the processing-js
library.  The details of this will be covered in Part 2.

By storing the object in the `obj` property on the generator, we can both step
through the generator and use a reference to the newly constructed object as 
necessary.  Here it's used in the helper function `_popAndStoreReturnValue`:

    private _popAndStoreReturnValue(value) {
        var frame = this.stack.pop();
        this._retVal = frame.gen["obj"] || value;
    }

One caveat of this approach is that if `classFn` is a generator which will be 
the case for all user defined constructors, it will be impossible to define a 
method on the class called `next` (or a property on the object).

## Code Transformation ##

Code transformations were performed use a variety or existing tools.  All of 
these projects are amazing and I couldn't have this project without them:

- [esprima] - parse JavaScript to Mozilla's [Parser API] format AST
- [estraverse] - tree walker for the AST (allows modifications
- [escope] - determine in scope variables
- [escodegen] - convert the AST back to JavaScript source

In order to simplify the transforms I added backlinks to parent nodes in the AST:

    estraverse.replace(ast, {
        enter: function(node, parent) {
            node._parent = parent;
        },
        leave: function(node, parent) {
            ...
        }
    });

The other thing that simplified the transform code was converting the bodyList
of statements in each Program and BlockStatement node to a linked list and then
then insert the yield statements into the linked list before converting it back
to an array.  I ended up creating a custom linked list library (part of [basic-ds])
which allowed easy access to to the nodes so that I could do the inserts while 
iterating.

Lastly, Firefox doesn't parse `yield` statements correctly so escodegen needed
to be modified to wrap all `yield` statements with parentheses.

## Sneak Peek ##

Next time we'll look at reccuring function calls and event handling.


[live-editor]:      https://github.com/Khan/live-editor
[debugjs]:          https://github.com/amasad/debugjs
[processing-js]:    http://processingjs.org/
[stepper]:          http://kevinb7.github.io/stepper/
[The Basics Of ES6 Generators]:     http://davidwalsh.name/es6-generators
[blog post]:        http://amasad.me/2014/01/06/building-an-in-browser-javascript-vm-and-debugger-using-generators/
[esprima]:          https://github.com/ariya/esprima
[Parser API]:       https://developer.mozilla.org/en-US/docs/Mozilla/Projects/SpiderMonkey/Parser_API
[estraverse]:       https://github.com/estools/estraverse
[escope]:           https://github.com/estools/escope
[escodegen]:        https://github.com/estools/escodegen
