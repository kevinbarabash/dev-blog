---
layout: post
title:  "TypeScript: Good, Bad, and Ugly"
date:   2014-12-11 14:16:23
categories: TypeScript
---

## Introduction ##

This is a tricky one.  There are quite a few things in JavaScript that feel
quite tedious to me when I'm typing them out that TypeScript makes a lot cleaner
with less noise.  On the other hand, once you start adding more type information
in and start working with external modules you'll end up doing things that you
didn't have to do before.  Initial this fells like a negative, but you have to
balance it with all the other productivity gains that are made.

I'm going to provide some examples of where TypeScript really shines and then
some areas of where it feels a little rough and what you can do to smooth things
over a bit.


## Good ##

Much of what I like about TypeScript is that it supports some of the new features
in ES6.  In particular the new syntax for classes, inheritance, auto-binding of
this, and the spread (rest) operator make JavaScript easier to write.

### Classes ###

The TypeScript class syntax support for default arguments makes this code cleaner
with less overhead.  This syntax matches ES6's class syntax.

__JavaScript__

    function Point(x,y) {
        x = x || 0;
        y = y || 0;
        this.x = x;
        this.y = y;
    }

    Point.prototype.mul = function (scalar) {
        this.x *= scalar;
        this.y *= scalar;
        return this;
    }

    // ...

__TypeScript__

    class Point {
        x: number;
        y: number;

        constructor(x = 0, y = 0) {
            this.x = x;
            this.y = y;
        }

        mul(scalar: number) {
            this.x *= number;
            this.y *= number;
            return this;
        }

        // ...
    }

### Inheritance ###

TypeScript really shines here.  It handles the esoteric details of setting up
the inheritance chain.  It also provides a `super` keyword which can be use
to call the superclass' constructor but call also be used to call methods on
the super class, e.g. `super.toString()` will call Shape's `toString` method
which more straight forward than `Shape.prototype.toString.call(this);`

__JavaScript__

    function Shape (color) {
        this.color = color;
    }

    Shape.prototype.toString = function () {
        return this.color + " shape";
    }

    function Circle (color, radius) {
        Shape.call(this, color);
        this.radius = radius;
    }

    Circle.prototype = Object.create(Shape.prototype);
    Circle.prototype.constructor = Circle;

    Circle.prototype.toString = function () {
        return this.color + " circle";
    }

__TypeScript__

    class Shape {
        color: string;

        constructor(color: string) {
            this.color = color;
        }

        toString() {
            return this.color + " shape";
        }
    }

    class Circle extends Shape {
        radius: number;

        constructor(color: string, radius: number) {
            super(color);
            this.radius = radius;
        }

        toString() {
            return this.color + " circle";
        }
    }

### Generics ###

The nice thing about generics is the type safety they provide.  If you're working
in a large code base with lots of data type it might not be immediately obvious
what properties and methods are available on items in an array.  Using generics
when paired with an editor that provides TypeScript intellisense can be huge
boon to developer productivity.  It can also help prevent mispelling property
names or insert instances of an unexpected type.

    interface Drawable {
        draw: () => void;
    }

    class Line {
        draw() {}
    }

    class Circle {
        draw() {}
    }

    class Vector {
        // no draw()
    }

    var scene = new Array<Drawable>();

    scene.push(new Line());
    scene.push(new Circle());
    scene.push(new Vector());   // error

You can also write `class Vector implements Drawable` which would raise an error
with the class definition because it doesn't have a `draw()` method.

### Fat Arrow ###

The fat arrow is new syntax coming in ES6 which you can use today.  It binds
`this` in functions the the local `this` instead of the global `this` which is
a common problem that leads to a lot of boilerplate.

__JavaScript__

    function Shape (canvas) {
        $(canvas).click(this.draw.bind(this));  // option 1
        $(canvas).click($.proxy(this,"draw"));  // option 2
        var self = this;                        // option 3
        $(canvas).click(function () {
            self.draw();
        });
    }

    Shape.prototype.draw = function () {
        // draw the shape
    }

__TypeScript__

    class Shape {
        constructor() {
            $(canvas).click(() => this.draw());
        }

        draw() {
            // draw the shape
        }
    }

The fat arrow can also take parameters.  The types of the parameters will match
those specified in the type of the callback.  This becomes very helpful when
using libraries like RxJS.

### ...rest (Spread Operator) ###


## Bad ##

- external modules, internal modules, and interfaces

## Ugly ##

- mixins

