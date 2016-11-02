---
layout: post
title: "Test Double with automatic cleanup"
date: 2016-11-01
categories: testing
---

I was having trouble with tests failing because I forgot to cleanup sinon stubs or td.replace.

I migrated everything to use testdouble only, always use `beforeEach`, and set up a global `afterEach(td.reset)`.

I found much more than just one problem. More like 10 or so. Tests that were passing because of side effects of previous tests that didn't clean up after themselves.

I consider myself a disciplined tester. If I can manage to make this many mistakes with cleanup, then probably many others would do the same or worse.

I previously loved to use `before` when possible to save on execution time. I now see that the speed is not nearly as valuable as the automatic cleanup.

Credit to Josh Egan for trying to convince me of this some time ago. It took me a while to learn the lesson myself.
