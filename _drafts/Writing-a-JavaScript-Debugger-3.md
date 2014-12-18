---
layout: post
title:  "Writing a JavaScript Debugger: Part 3"
categories: javascript debugging
---

## Stack Frames ##


## Local Variables ##


## Future Work ##

Support callbacks for various functions:

- addEventListener
- setTimeout/setInterval
- XMLHttpRequests
- jQuery's on, bind, etc.
- Array.prototype.forEach/map/filter/reduce/etc.
- requestAnimationFrame

Provide more accurate execution location information so that if there are 
multiple method calls on a single line, each can be highlighted before it's 
executed as opposed to highlighting the whole line and not knowing which part
we're paused on.

Provide a complete list of in scope variables (or at least in scope variables
that a being used).  The current debuggers will provide a list of locally defined
variables as well as variables caught in the closure and globals.  If a user 
wants to inspect a global that's being used it can be quite tedious to select
it a from a list of all the globals when there are hundereds to select from from.

