---
layout: post
title:  "TypeScript: Why and How"
date:   2014-12-11 14:16:23
categories: TypeScript
---

## Introduction ##

I started tinkering with TypeScript over a year ago when it was still around
version 0.8.  WebStorm (my editor of choice) had just announced support for it
in one of its EAP releases.  I played around with it a little bit but didn't
use it for anything serious until 1.0 was released.

Once 1.0 was out, I took another look at it.  I was managing a project working
on a relatively large JavaScript code base (approximately 30,000 lines at the
time).  One of the issues we had was that it was taking some of the new team
members a while to ramp up.  Another issue was that every once in a while we
were getting bit bugs involving type coersion.

We investigated TypeScript to see if it could help with these problems.  We had
several criteria:

- easy for existing JavaScript developers to get up to speed
- supports incremental migration
- easy to revert back to JavaScript if something goes drastically wrong
- tooling support (more accurate code completion, refactorings, navigation, etc.)
- eliminates a certain class of bugs due to JavaScript's dynamic typing

Although we weren't able to convert all of the code base to TypeScript (mainly
because of higher priority work) we did get about 20% converted.  As we were
doing these conversions a couple of things stuck out at me:

- we ended up finding bugs (needs examples)
- code that was hard to convert as often bad code (needs examples)

## The Good ##

### Incrementalism ###

If you have a JavaScript code base and you want to start using TypeScript with
it you can use TypeScript with just the new modules while leaving the remaining
ones as JavaScript.  You can convert some JavaScript modules to TypeScript a few
at a time.  One thing we discovered though, was that if you wanted accurate
type information it was best to convert JavaScript modules with no dependencies
first and then work your way out to modules whose dependencies had all been
converted.

There's another type of Incrementalism at work in TypeScript and that's the type
system itself.  You don't have to put in any type declarations yourself.  If you
don't they are implicitly given the type "any".  TypeScript has a powerful type
inference engine and often figure out what type an variable is and what methods
it might have.  You can optional add types to paramters to specify what's acceptable.

Also, any of the syntax which TypeScript adds is opt in as well.

### Community ###

TypeScript also lets you define .d.ts (declaration) files that describe
the structure of your JavaScript interfaces without having to convert the code
itself.  This means you can use libraries like jQuery without having to rewrite
jQuery.  The TypeScript community has created hundreds of the .d.ts files for
many libraries.  They reside in a github project called [DefinitelyTyped].

There's even a "package manager" for adding these files to your project.  It's
called tsd and it works kind of like npm or bower.  It's a great way to grab
the files you need without having to clone the whole DefinitelyTyped repo or
download them by hand.


## TODO ##

- DefinitelyTyped [done]
- tsd [done]
- tslint
- build tools
    - tsify
    - grunt/gulp tsc
    - tsc's built-in watcher
    - tsc + watchify


DefinitelyTyped:        [https://github.com/borisyankov/DefinitelyTyped]
