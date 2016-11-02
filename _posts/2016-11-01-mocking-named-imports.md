---
layout: post
title: "Mocking Named Imports in Node.js"
date: 2016-09-17
categories: testing
---

## foo.js

{% highlight javascript %}
export function sayFoo() {
  return 'foo is the best'
}
{% endhighlight %}

## bar.js

{% highlight javascript %}
import { sayFoo } from './foo'

export function bar() {
	return sayFoo()
}
{% endhighlight %}

## Bad way to test bar.js

The following works, but it is because of a babel transpilation implementation detail.

{% highlight javascript %}
import * as foo from './foo'
import { bar } from './bar'

describe('bar', () => {
  let result

  beforeEach(() => {
    td.replace(foo, 'sayFoo')
    td.when(foo.sayFoo()).thenReturn('test')
    result = bar()
  })

  it('should return the result of sayFoo', () => {
    expect(result).to.equal('test')
  })
})
{% endhighlight %}

What should happen here is that bar has a pointer directly to sayFoo before our test setup executes. When our test setup executes, it changes pointer foo.sayFoo to point to a mock implementation, but that doesn't affect bar's pointer which still points to the real sayFoo.

So why does this work? If we go to http://babeljs.io/repl/ and paste the contents of `bar.js`, we get the following:

{% highlight javascript %}
'use strict';

Object.defineProperty(exports, "__esModule", {
 value: true
});
exports.bar = bar;

var _foo = require('./foo');

function bar() {
  return (0, _foo.sayFoo)();
}
{% endhighlight %}

Whaaaaat? Rather than destructuring the module and making a pointer `sayFoo` that points directly to the function, it imports the entire module as an object and updates all your calls to sayFoo to be via that imported object.

Let's change our implementation of bar.js as follows:

## bar.js with destructuring

{% highlight javascript %}
const { sayFoo } = require('./foo')

export function bar() {
  return sayFoo()
}
{% endhighlight %}

And now our test fails because it calls the real implementation of sayFoo, not the mocked one.

Why is this a problem? Because when we have a real implementation of javascript modules in the near future and you want to start using that, you may find that suddenly you have a ton of tests failing. :(

My suggestion, don't rely on this bebel implementation detail. I suggest the following pattern for testing bar.js instead:

## A better way to test bar.js

{% highlight javascript %}
import proxyquire from 'proxyquire'

describe('bar', () => {
  let bar, result

  beforeEach(() => {
    bar = proxyquire('./bar', {
      './foo': {
      	sayFoo: td.when(td.function('sayFoo')()).thenReturn('test'
      }
    }).bar
    result = bar()
  })

  it('should return the result of sayFoo', () => {
    expect(result).to.equal('test')
  })
})
{% endhighlight %}
