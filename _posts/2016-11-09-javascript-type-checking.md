---
layout: post
title: "JavaScript Type Checking"
date: 2016-11-09
categories: javascript
---

# Intro

Type Checking in JavaScript is a controversial topic. There was at one point a proposal to add typed objects to JavaScript. This proposal was [abandoned](https://github.com/tc39/ecma262/commit/02455e5e2964f62b13818c6fd23289381ecafdf8) but there are [continuing efforts to work on a new proposal](https://github.com/nikomatsakis/typed-objects-explainer). [Flow](https://flowtype.org/) and [TypeScript](https://www.typescriptlang.org/) are gaining a lot of popularity. If you are interested in type checking in JavaScript, you have options.

# Why?

Many who have adopted JavaScript did so as an escape from strongly typed languages like Java and C#. Languages like these which are not only strongly typed, but also heavily object oriented can sometimes require a lot of ceremony. JavaScript is very loose and can be used in a purely functional way. So why would we want to add typing?

 - Safety
 - Toolability
 - Documentation

# Why not?

The need for type checking grows with larger projects (think monoliths). Trying to change the name of a function is very difficult when that function is used in hundreds of places and can't be traced. With strongly typed languages, this can be an easy and reliable automated process, usually facilitated by an IDE. Some argue that if you need typing in your project, the project has become too large/monolithic.

Also, the only two solid typing options for JavaScript that I'm aware of are TypeScript and Flow. Migrating a large project to either of these is disruptive. A transpilation step is necessary (unless you use flow with comment style annotations) and typically requires many code changes to make your project compliant.

# I like types, but want something less disruptive

Ideally, I want the best of both worlds. I think types are super helpful, but I don't want them forced on me. I want to add them optionally where I think they add value. I dislike transpilation. For code that will run in the browser, it is worth the pain, but for my server-side node projects, I find the syntax supported by Node >= 6 is sufficient and therefore don't want to introduce a transpilation layer.

*Why I dislike transpilation*

 - Heavy (TONS of packages)
 - Makes debugging harder (have to get source maps set up just right, can't just look at plain source)
 - Intermediate step added to just about anything (running code, webpack/gulp stuff, etc)

# Sidebar: Flow vs TypeScript

Useful comparison of Flow and TypeScript
https://djcordhose.github.io/flow-vs-typescript/2016_hhjs.html#/

TypeScript
 - Always transpilation
 - Also == Close-ish to ES6/7/..., but never perfect. Not ACTUALLY ES6.
 - Can get type checking of 3rd party stuff with d.ts files. It's a pain, but at least it's an option. I think
   you could skip this by configuring not strict.

Flow
 - Can do some type checking w/out writing any types (no transpilation)
 - Flow can use comments for annotations so that transpilation is necessary starting with 0.4.
 - You can start adding in type assertions, but then you need to transpile (w/ babel)
 - But the transpilation is super simple. Line numbers should match up w/out source maps. Footprint is probably small.
 - Seems like it is not capable of checking 3rd party stuff.
 - How to check complex data types?
 - Does Flow have WebStorm support?
 - Online playground supports Class properties. Not supported by node? Need Babel?
 - Only tests a file if it has the // @flow comment at the top (configurable in .flowconfig?)
 - Flow doesn't let me set properties on a class that weren't defined as class attributes, which Node doesn't support w/out Babel!
 - Flow has a weak mode that makes it easier to add to an existing project. It is advised to start here, then add types gradually, then switch to normal mode.
 - I tried to run flow on an existing project and got a TON of errors on stuff in node_modules and can't find a way to ignore. I only added `// @flow weak` to one file and couldn't get it to test only that one file.

# Enter JSDoc

JSDoc is a classic way to document your JS code and it *almost* provides what I need. It provides the missing metadata that is needed to make my code more type safe and toolable. It provides much more context to consumers of my code. However, JSDoc by itself won't tell you if you passed a string where you should have passed a number. For that you need a tool that will leverage JSDoc to provide type checking.

## WebStorm

WebStorm has excellent contextual suggestions and type checking for JavaScript and uses JSDoc to enhance this output.

 - Type `Ctrl + q` to get a full JSDoc explanation of a property
 - Signatures provided while typing
 - Violations noted inline with a level of your choice (Weak Warning by default)
 - WebStorm 2016.2 gets confused by destructured import syntax. Fixed in 2016.3. Example: `const {doIt} = require('./foo')`
 - To get tooling for npm installed packages, you must not exclude your node_modules directory. This means that node_modules will show up in project-wide searches, open file dialog, symbol matching, etc.
   - You can exclude node_modules from search results without marking the directory as excluded by creating a custom search scope. In that scope, recursively include your whole project and then recursively exclude node_modules.
   - You might also choose to go ahead and mark node_modules as excluded and then mark specific packages as Not Excluded.
 - You can choose the severity level failed inspections when you don't conform to the types specified in your JSDoc. The inspections you want to set are `Signature mismatch` and `Type mismatch`, both under `JavaScript`.

## Other tooling?

The biggest problem I have with leaning on JSDoc for typing is that I am highly dependent on WebStorm to provide the critical integration. I LOVE that my editor will provide all the information I need inline in my editor. However, I wish I could have a command line process that I could run on a continuous integration server or a git hook that ensures I didn't break any defined types.

# Examples using JSDoc + WebStorm

{% highlight javascript %}
/**
 * Does the awesomeness.
 *
 * @param {Object} opts Options
 * @param {string} opts.name
 * @param {number} [opts.age]
 * @returns {% raw %}{{foo:number}}{% endraw %}
 */
function foo(opts) {
  return {foo: 6}
}

/** @type {number} */
let it = 6

foo({name: 'hello', age: 6})
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
/** @typedef {% raw %}{{x:number, y:number}}{% endraw %} point */
{% endhighlight %}
