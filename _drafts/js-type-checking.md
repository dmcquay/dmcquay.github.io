---
layout: post
title: "JavaScript Type Checking"
date: 2016-09-17
categories: javascript
---

Useful comparison of Flow and TypeScript
https://djcordhose.github.io/flow-vs-typescript/2016_hhjs.html#/

I dislike transpilation. Why?
 - Heavy (TONS of packages)
 - Makes debugging harder (have to get source maps set up just right, can't just look at plain source)
 - Intermediate step added to just about anything (running code, webpack/gulp stuff, etc)

TypeScript
 - Always transpilation
 - Also == Close-ish to ES6/7/..., but never perfect. Not ACTUALLY ES6.
 - Can get type checking of 3rd party stuff with d.ts files. It's a pain, but at least it's an option. I think
   you could skip this by configuring not strict.

Flow
 - Can do some type checking w/out writing any types (no transpilation)
 - You can start adding in type assertions, but then you need to transpile (w/ babel)
 - But the transpilation is super simple. Line numbers should match up w/out source maps. Footprint is probably small.
 - Seems like it is not capable of checking 3rd party stuff.
 - How to check complex data types?
 - Does Flow have WebStorm support?
 - Online playground supports Class properties. Not supported by node? Need Babel?
 - Only tests a file if it has the // @flow comment at the top (configurable in .flowconfig?)
 - Flow doesn't let me set properties on a class that weren't defined as class attributes, which Node doesn't support w/out Babel!

TODO
 - Can either option detect when I mock something with the wrong parameters?

Maybe with yarn I won't mind the footprint of babel anymore?
