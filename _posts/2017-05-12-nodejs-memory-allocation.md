Nodejs uses v8 under the hood.
v8 manages memory using a heap
the heap consists of two chunks of memory (old and new, also called large and small)
Default max old size is ~1.5 G. Max on 64bit machines is 1.7G. Max on 32bit is 512M.
You can control max old size with --max_old_space_size=X (in MB)


--max_new_space_size (in kBytes) - i think this is old

--max-executable-size the maximum size of heap reserved for executable code (the native code result of just-in-time compiled JavaScript).
--max-old-space-size the maximum size of heap reserved for long term objects
--max-semi-space-size the maximum size of heap reserved for short term objects

References:

- https://github.com/nodejs/node/wiki/Frequently-Asked-Questions
- http://stackoverflow.com/questions/27175301/what-are-v8-old-space-and-new-space
- http://erikcorry.blogspot.ru/2012/11/memory-management-flags-in-v8.html (good, but old. i think new_space has changed)
- http://stackoverflow.com/questions/30252905/nodejs-decrease-v8-garbage-collector-memory-usage (references updated new_space flags, all in MB now)
- https://github.com/nodejs/node/blob/master/deps/v8/src/api.cc#L822 (v8 defaults from souce code -- apparently somewhat dynamic based on device memory)
