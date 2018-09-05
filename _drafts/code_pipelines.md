About two and a half years ago I was working at Pluralsight and writing some of the best code of my life. The quality was great. It was well tested. I felt proud.

Some months later, our team began discussing the cost of our testing suite in terms of development effort. The problem was that two thirds of our code was tests and two thirds of the tests were mocks[1]. We agreed that if this was the only way to get the quality we desired, then it was worth it. No regrets. Bugs were exceedingly rare. We spent nearly all our team working on new features and little to no time wasted on bugs or brittle systems. It was wonderful.

But we wondered if there was a better way? Was there a different approach where we could get the same quality without so much mocking code. It was at this time that I began a long journey into Functional Programming, Functional Core Imperative Shell, Effects as Data, and other approaches for writing code in such a way that fewer mocks are needed for testing. Recently I got some insights from an Architect at Pluralsight (Tim Cash) that really helped me bring it all together in a way I think I can buy into and recommend to others. I hope that Tim will publish his ideas, but for now I will do my best to share my interpretation of it, along with a concrete example from a project I am currently working on.

## Code Pipelines in JavaScript

- Maximize pure functions by isolate impure functions
- Structure code as pipelines instead of hierarchies
- Return errors as values
- Logging and monitoring are impure
- Creatively avoid special handling of edge cases
- How to test your coordination functions
- Runtime abstration
- Team master and ownership > clever code
- Decouple from protocol (e.g. HTTP)

[1] This is a rough estimate from my recollection. I did not take the time to do a code analysis for this post.