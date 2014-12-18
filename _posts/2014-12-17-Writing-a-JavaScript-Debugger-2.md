---
layout: post
title:  "Writing a JavaScript Debugger: Part 2"
date:   2014-12-17 14:16:23
categories: javascript debugging
---

This is part two in a three part series detailing the implementation of a 
JavaScript debugger for using with Khan Academy's [live-editor].  Here's a link
to [part 1].

## Event Loop ##

JavaScript is single threaded.  This can make it easier to write software because
you know that the function cannot be interrupted by an event.  When debugging,
if you're stepping through a function you're reguarly interrupting its execution
to return control to the browser so that it can process events and keep the UI
responsive.

In order to maintain the single-threaded non-reentrant semantics of JavaScript
we need to simulate a our own event loop. The Scheduler is responsible for 
maintaining a queue of pending functions to be executed.  When the debugger is 
asked to step (or continue) it will ask the its scheduler instance for the 
current stepper and then forward the stepper the appropriate action. When the 
stepper finishes it will signal the scheduler it is done and the scheduler will 
remove it.

The stepper signals that it's done by calling its `doneCallback` which the
Scheduler wrapped when the Stepper was added as a task.  Here are the relevant
methods from scheduler.ts:

    addTask(task) {
        var done = task.doneCallback;
        task.doneCallback = () => {
            this.removeTask(task);
            done();
        };
        this.queue.push_front(task);
        this.tick();
    }
    
    private removeTask(task) {
        if (task === this.currentTask()) {
            this.queue.pop_back();
            this.tick();
        } else {
            throw "not the current task";
        }
    }

I tried a few approaches before settling on this.  I didn't want the Scheduler
or Stepper to have explicit knowledge about each other.  Also, the Debugger 
needs to be notified when the main function completes so the Scheduler can't 
simply set the `doneCallback` on the Stepper isntance.  One alternative I 
investigated was using an `EventEmitter`, but that that introduced another 
dependency that didn't really add much.

The `tick()` method is called anytime a new task (Stepper) is added or completed.  
It checks if the current task has been started.  If it hasn't it starts it.  It
does this asynchronously so that the UI will remain responsive.
 
    private tick() {
        setTimeout(() => {
            var currentTask = this.currentTask();
            if (currentTask !== null && !currentTask.started) {
                currentTask.start();
                this.tick();
            }
        }, 0);  // defer execution
    }

## Draw Loop ##

Processing-js has a method called `loop` which sets up a `setTimeout` based loop 
which calls `draw` after each timer expires.  Because the debugger doesn't have
support for handling `setTimeout` calls where the callback is a generator we 
have to set up our own draw loop.  This is done in the `onMainDone` callback in
ProcessingDebugger.

    // ProcessingDebugger::onMainDone
    var draw = this.context.draw;
    
    if (draw !== emptyFunction) {
        this._repeater = this.scheduler.createRepeater(
            () => this._createStepper(draw()),
            1000 / 60
        );
        this._repeater.start();
    }

This callback provides an opportunity for subclasses of Debugger to do any
setup work before other functions start executing.  In this case we're asking
the Scheduler to create a Repeater which will queue new instances after a 
timer expires.  It's passed a function which creates a task.  In this case it's
an Arrow function which returns a new Stepper instance.  It returns an object
with a `start()` method which calls `repeatFunc()`, a private function, which
looks like this:

    function repeatFunc() {
        if (!_repeat) {
            return;
        }
        var task = createFunc();
        var done = task.doneCallback;
        task.doneCallback = function () {
            if (_repeat) {
                setTimeout(repeatFunc, _delay);
            }
            done();
        };
        _scheduler.addTask(task);
    }

The interesting part of this function is that it only creates a new timer after
when the `doneCallback` is called.  Since the callback is only called when a 
task (Stepper) completes and the Scheduler only add a new task when the previous
one has completed, it prevents repeating tasks from piling up when the debugger
is paused.

## Event Handling ##

Queuing of calls to event handlers is quite a bit easier.  If any of the special
event handling methods in processing-js are defined in the main function when
its done running, they are replaced by a function which will queue the 
corresponding generator.  The function `queueGenerator` adds the generator to 
the debugger's scheduler.

    // ProcessingDebugger::onMainDone
    ProcessingDebugger.events.forEach(name => {
        var eventHandler = this.context[name];

        if (_isGeneratorFunction(eventHandler)) {
            if (name === "keyTyped") {
                this.context.keyCode = 0;    // preserve existing behaviour
            }

            this.context[name] = () => {
                this.queueGenerator(eventHandler);
            };
        }
    });
    
    // ProcessingDebugger
    static events = [
        "mouseClicked", "mouseDragged", "mousePressed", "mouseMoved", "mouseReleased",
        "keyPressed", "keyReleased", "keyTyped"
    ];

Actually handling the events coming from the browser and making sure they are
delivered to processing-js in the correct order with the correct properties and 
timing is the complicated part.  

Processing-js listens to `mousemove`, `mousedown`, `mouseup` to determine when 
the mouse was clicked and dragged.  It also updates variables `mouseX`, `mouseY`,
`pmouseX`, and `pmouseY`.  If the user sets a breakpoint on `mouseClicked` and 
the breakpoint gets hit because the user clicked, the use should be able to 
move they're mouse so that can operator the debugger controls without worrying
about changing the values of `mouseX` et al.

This means that we have to filter the events going to the program being debugged.
I wanted to avoid modifying processing-js so instead I put the programming being
debugged in an iframe and create an [iframe-overlay] which filters events.

In the previous post, I mentioned that I thought the iframe used by debugjs was
interferring with setting the fill color.  This is because debugjs tries to use
a direct reference to iframe's contentWindow.

In our case the debugger is inside the iframe along with the code that it's 
debugging.  Debugger control messages are communicated using postMessage via a 
wrapper called [poster].  Poster also wraps postMessage between windows and 
web workers.

The iframe-overlay library provides two methods:

- createOverlay(iframe)
- createRelay(element)

The `createOverlay(iframe)` function is called in the parent DOM context to 
create a `<span>` which covers the iframe and intercepts the events.  The
`createRelay(element)` function is called in the iframe's DOM context and passed
element which you want to retrigger events on.

The interesting thing about the overlay is that it has a `pause()` method which 
allows the overlay to temporarily stop forwarding events.  All events that the
overlay receives while it's paused are queued.  When the overlay resumes 
forwarding events, it will send the queued events in the correct order.

It will also update the timestamps so that events are delivered with the same 
inter-event delays that they were receieved with.  I thought that maintaining
the delays would help in avoiding [heisenbug]s.

## Cleanup ##

Before restarting the debugger all of the event handlers and `draw` function
are removed from the context.   

    onMainStart() {
        if (this._repeater) {
            this._repeater.stop();
        }

        // reset all event handlers
        ProcessingDebugger.events.forEach(event => this.context[event] = emptyFunction);

        // reset draw
        this.context.draw = emptyFunction;
    }

## Processing Edge Cases ##

In the process of manual testing I ran into an issue where the `keyCode` was
being incorrectly reported in the `keyPressed` event handler.  The reason for 
this was because processing-js's `simulateKeyTyped` function assumes that the
`keyPressed()` and `keyTyped()` methods will be executed synchronously.  

    function simulateKeyTyped(code, c) {
        pressedKeysMap[code] = c;
        lastPressedKeyCode = null;
        p.key = c;
        p.keyCode = code;
        p.keyPressed();
        p.keyCode = 0;    // clears keyCode before we finish executing keyPressed
        p.keyTyped();
        updateKeyPressed();
    }

This assumption doesn't hold when debugging and there's nothing we can do about
this without modifying the processing-js code.  Thankfully, this is the only
thing that required modification.  The other option would have to be to convert
some or all of the methods defined by processing-js to generators and then 
step through them.  This would've added a more complexity than it was worth.

## live-editor ##

Live-editor keeps track of object instances between successive runs.  It does 
this by using a special function called `applyInstance` to create objects as 
opposed to using `new`.  This method gives each object an id that's based on 
the constructors name and the arguments that were passed to the constructor.

The debugger replaces all instances of `new` with calls to `__instantiate__`, 
but `applyInstance` was being called first an replacing all the `new` calls 
before the debugger could get to them.  Because `applyInstance` doesn't know
how to deal with generators I decided it would be easier to notify the live-editor
whenever a new object had been created and give it all the information it needed
to track object instances.

I extracted all of the object tracking code from `applyInstances` into a method
called `newCallback` which is called by either `applyInstances` or the debugger
whenever an instance is created.

    newCallback: function (classFn, className, obj, args) {
        // Make sure a name is set for the class if one has not been
        // set already
        if (!classFn.__name && className) {
            classFn.__name = className;
        }

        // Point back to the original function
        obj.constructor = classFn;

        // Generate a semi-unique ID for the instance
        obj.__id = function() {
            return "new " + classFn.__name + "(" +
                this.stringifyArray(args) + ")";
        }.bind(this);

        // Keep track of the instances that have been instantiated
        if (this.instances) {
            this.instances.push(obj);
        }
    }

Please see part one for the definition of `__instantiate__`.

## Next Time ##

In part 3 we'll look at stack frames, inspecting local variables, and future 
work.


[live-editor]:          https://github.com/Khan/live-editor
[part 1]:               /dev-blog/{% post_url 2014-12-15-Writing-a-JavaScript-Debugger-1 %}
[iframe-overlay]:       https://github.com/kevinb7/iframe-overlay
[poster]:               https://github.com/kevinb7/poster
[heisenbug]:            http://en.wikipedia.org/wiki/Heisenbug
