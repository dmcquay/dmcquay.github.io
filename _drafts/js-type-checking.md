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


Update:
 - Flow can use comments for annotations so that transpilation is necessary starting with 0.4.
 - Flow has a weak mode that makes it easier to add to an existing project. It is advised to start here, then add types gradually, then switch to normal mode.
 - I tried to run flow on narwhal and got a TON of errors on stuff in node_modules and can't find a way to ignore. I only added `// @flow weak` to one file and couldn't get it to test only that one file.


What about just using JSDoc?
 - Will WebStorm read the JSDoc and give us better tooling?
 - Typescript supposedly will read and use JSDoc. Could we use JSDoc + Typescript as a linter instead of a compiler?


{% highlight javascript %}
/**
 * Does the awesomeness.
 *
 * @param {Object} opts Options
 * @param {string} opts.name
 * @param {number} [opts.age]
 * @returns {{foo:number}}
 */
function foo(opts) {
  return {foo: 6}
}

/** @type {number} */
let it = 6

foo({name: 'hello', age: 6})
{% endhighlight %}

In WebStorm, destructured require syntax interferes with WebStorm's ability to trace the types:

{% highlight javascript %}
import {doIt} from './foo' // works
const foo = require('./foo') // works, but you have to call as foo.doIt()
const {doIt} = require('./foo') // doesn't work
{% endhighlight %}

Complex types:

{% highlight javascript %}
/**
 @typedef {Object} point
 @property {number} x The x coordinate.
 @property {number} y The y coordinate.
 */
{% endhighlight %}

or shorthand:

{% highlight javascript %}
/** @typedef {{x:number, y:number}} point */
{% endhighlight %}




TODO:
 - Document the use cases where WebStorm can't follow your code (e.g. destructured require statements)
 - Find a command line checker/linter that does what WebStorm does. Ideally identical. spy-js?
