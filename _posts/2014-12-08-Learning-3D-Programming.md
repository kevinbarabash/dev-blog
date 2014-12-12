---
layout: post
title:  "Learning/Teaching 3D Programming (is hard)"
date:   2014-12-08 14:16:23
categories: 3D education programming
---

## Introduction ##

I've been doing 3D programming on and off for a long time.  I started out using
QBasic on an old 486 doing naive calculations for a rotating wireframe cube.
Later, it was C and OpenGL from the Red Book.  More recently, it's WebGL and
shader programming and GLSL.js and THREE.js on top of that.

I've made a lot of mistakes.  3D is hard.  There are so many times I've hit "run"
and been left staring at a blank screen scratching my head.  Getting your first
triangle is definitely a lot easier thanks to frameworks like THREE.js, but when
things go awry it can be difficult to pin down what's going wrong.

## Why 3D? ##

In a previous life I was a high school teacher.  I taught a variety of courses
in the computer lab including intro programming.  Some of the students were quite
keen to learn 3D programming because they wanted to make their own games.  Most
of the games they played were 3D games on consoles and they wanted to do something
similar.

3D graphics represent the real world and perhaps learning about how to create
programs that render 3D objects might give students some insights into the
physcial world.

Lastly, software is being used more and more to interact with the real world
through robots of various sorts and 3D printers.  Knowledge of 3D programming
can help with learning to program these types of devices.

## 3D on the Web ##

WebGL is essentially a port OpenGL ES 2.0 to the browser.  It's relatively modern
in the shaders as opposed to a fixed pipeline which is how OpenGL used to work
a long time ago.  This provides a lot of flexibility to developers, but getting
your first triangle on to the screen though involves a lot of steps.  You'll need
to:

- create a WebGL context
- write vertex/fragment shaders
- compile the shader program
- create buffers with geometry, color, etc. data (this is actually two steps)
- bind them to attributes
- set up projection/view matrix
- bind uniforms for the project/view matrix
- issue drawing commands

Libraries like THREE.js help by doing a lot of the heavy lifting for you and
provide escape hatches if you ever need more control, e.g. you can write your
own shaders with THREE.js.  THREE.js provides a scene graph API which is much
simpler by there's still a fair amount of setup that needs to be done:

- create a WebGL renderer
- set up camera
- create a Geometry object
- create a Material object
- create a Mesh object that combines the Geometry and Material
- add the mesh to the scene
- render the scene

## What Can Go Wrong ##

While libraries like THREE.js are a huge improvement over WebGL's API, there are
still lots of things that can go wrong:

- the camera is pointing the wrong way/the object is out of view
- the clipping plane is excluding the object
- the camera is inside the object
- the camera is too far way from the object so it's too small to see
- the polygon normals are facing the wrong way
- the material requires lights
- the lights aren't in the right location/pointing the right way
- the lights aren't bright enough

That's why I usually start with a working example which I then modify to fit the
project goals.  Although this helps during the getting started stage, once you
start creating custom geometry or changing the camera from the initial settings
it can be hard to figure out why something isn't working as expected.

## Fumbling Around in Darkness ##

I usually end up doing a lot of guess-and-check to solve the problem.  Change
the clipping planes to be really large until I can see something.  Change the
size of the object until I can see something.  Move the camera around until I
can see something.  Double check that the vertices for the object's faces are
in counter-clockwise order.

I started working on a project to code interactive platonic solids using Khan
Academy's live-editor.  I love their CS curriculum and want to create some
programs that could be useful examples for Math but might also give students
a basis for them to created their own interactive 3D programs.

The Math related goals included providing a way to:

- toggle faces, edges, and vertices
- adjust opacity
- show dashed lines hidden lines when the faces were semi-opaque

The live-editor currently doesn't have any built-in support for 3D so I ended
up writing code for all of the calculations.  I kept it simple by using an
orthographic projection where the z coordinate is discarded on rendering.  The
view is also translated so the origin is in the center of the canvas and the
y-axis is flipped so that y gets bigger going up.  This is the same thing that
Peter Collingridge does in his tutorials on Khan Academy [creating-3d-shapes].

This brings up an interesting point about coordinate systems.  Most 2D graphics
libraries use the upper-left as the origin with positive y pointing down.  While
I understand why this choice is common, it becomes bothersome once you start
trying to do apply math that you learned in school, especially trigonometry.

Because of this confusion, I decided to flip the y-axis.  As for centering the
origin, a common activity when viewing 3D objects is to orbit the object.  If
the object is centered at the origin which is at the center of your view this
becomes much easier.

In order to keep the implementation "simple", I relied on the fact that all of
the platonic solids were convex.  This meant that when the solid is full opaque,
edges which are adjacent to two backfacing polygons can be ignored (or drawn
with dashes).  Similar reasoning allows certain vertices to be discarded.  All
of these calculations required knowing the normal.

The normal is also useful in shaders for lighting and other effects.  In many
3D modelers there's a function to toggle the normals so I added that to my todo
list.

- avoiding gimbal lock

## Finding the Light Switch ##

As I was looking through the examples on [creating-3d-shapes] I noticed that one
of the examples had the vertices label with their index.  The indices of vertices
are important because other primitives such as edges and faces are constructed
by specifying the index of each of their vertices.

That got me thinking.  What if I started out with just the vertices and then was
able to add faces and edges as I rotated the object.  The live-editor allows you
to make changes to your program and see the results immediately.

TBC

## Activities ##

- connect the dots
- Platonic duals
- subdivision surfaces
- terrain generation with noise


[creating-3d-shapes]: https://www.khanacademy.org/computing/computer-programming/programming-games-visualizations/programming-3d-shapes/a/creating-3d-shapes
