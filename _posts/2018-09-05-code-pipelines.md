---
layout: post
title:  "Code Pipelines in Javascript"
date:   2018-09-05 08:30:00
tags:   Javascript TDD Testing
---

About two and a half years ago I was working at Pluralsight and writing some of the best code of my life. The quality was great. It was well tested. I felt proud.

Some months later, our team began discussing the cost of our testing suite in terms of development effort. The problem was that two thirds of our code was tests and two thirds of the tests were mocks[1]. We agreed that if this was the only way to get the quality we desired, then it was worth it. No regrets. Bugs were exceedingly rare. We spent nearly all our team working on new features and little to no time wasted on bugs or brittle systems. It was wonderful.

But we wondered if there was a better way? Was there a different approach where we could get the same quality without so much mocking code. It was at this time that I began a long journey into Functional Programming, Functional Core Imperative Shell, Effects as Data, and other approaches for writing code in such a way that fewer mocks are needed for testing. Recently I got some insights from an Architect at Pluralsight (Tim Cash) that really helped me bring it all together in a way I think I can buy into and recommend to others. I hope that Tim will publish his ideas, but for now I will do my best to share my interpretation of it mixed with a few additions of my own.

## How to write quality code that is easy to test in JavaScript

- Structure code as pipelines instead of hierarchies
- Maximize stateless functions by isolating I/O
- Return errors as values
- Logging and monitoring are impure
- Creatively avoid special handling of edge cases
- How to test your coordination functions
- Creating a runtime

## Structure code as pipelines instead of hierarchies

We should have a top level function which I will refer to as a coordinator function. It is the topmost interface to your application except for the protocol. For example, if you are handling an HTTP request and sending an HTTP response, the coordinator function would not know anything about HTTP, but it will return any information you need to build your HTTP response.

In your coordinator functions, only two things should happen:

1. Call a stateless function that does some kind of transform.
2. Call a stateful function (such as a DB query) and then check if it failed.

```
function coordinatorFunction(props) {
    const queryCommand1 = buildQueryCommand1(props); // stateless transform
    const result1 = executeQuery(queryCommand); // stateful operation (I/O)
    if (isFailure(result1)) { // check for failure
        logger.error(getError(result1))
        return result1;
    }
    const payload1 = getPayload(result1);

    const queryCommand2 = buildQueryCommand2(props); // stateless transform
    const result2 = executeQuery(queryCommand2); // stateful operation (I/O)
    if (isFailure(result2)) { // check for failure
        logger.error(getError(result2))
        return result2;
    }
    const payload2 = getPayload(result2);

    const finalResult = buildResult(payload1, payload2);
    return success(finalResult);
}

function httpRoute(req, res) {
    const props = buildPropsFromReq(req);
    const result = coordainatorFunction(props);
    if (isFailure(result)) {
        logger.error(getError(result));
        if (getError(result).errorType === 'unauthorized') {
            res.status = 400;
        } else {
            res.status = 500;
        }
    } else {
        res.status = 200;
        res.body = getPayload(result);
    }
}
```

## Maximize stateless functions by isolating I/O

We often mix business logic with I/O or other stateful functions. Because of this, the business logic is harder to test. Here's an example:

```
function getTransactions(accountId) {
    const sql = 'SELECT * FROM transactions WHERE account_id = $1';
    const result = await db.query(sql, [accountId]);
    return result.rows.map(row => {
        return {
            id: row.id,
            amount: row.amount,
            createdAt: row.created_at
        };
    });
}
```

We are really doing three things here:

1. Building a command
2. Executing the command (I/O)
3. Transforming the result

Items 1 and 3 are very easy to test if they were not coupled with 2. Let's change that.

```
// This is now our coordinator function.
function getTransactions(accountId) {
    const command = buildSelectionTransactionsCommand(accountId);
    
    const result = await db.query(command);
    if (isFailure(result)) { // check for failure
        logger.error(getError(result))
        return result;
    }
    const rows = getPayload(result);

    const dtos = rows.map(buildTransactionDtoFromDbRow);

    return success(dtos);
}

function buildSelectTransactionsCommand(accountId) {
    return {
        sql: 'SELECT * FROM transactions WHERE account_id = $1',
        params: [accountId]
    };
}

function buildTransactionDtoFromDbRow(row) {
    return {
        id: row.id,
        amount: row.amount,
        createdAt: row.created_at
    };
}
```

Now `buildSelectTransactionsCommand` and `buildTransactionDtoFromDbRow` are separated from the I/O and can easily be tested.

## Return errors as values

You've probably wondered `success`, `failure`, `isFailure`, `getError` and `getPayload` functions are. Here I am using Failables to reprensent my errors and these are utility functions that help me work with these objects in an abstract way. I don't want to convince you to use Failables necessarily, let alone this crude impelementation of them. Rather, I want to advocate for passing errors as values instead of throwing them.

Throwing errors is convenient because you can choose to ignore errors and "catch" them several layers up. It is bad for the same reason. Especially in Javascript since there is no type system to make you deal with thrown errors.

A major reason for the first principle **Structure code as pipelines instead of hierarchies** is so that there are no surprises. We don't want surprising state manipulations to happen one or more layers beneath us. We want all such manipulations to happen in plain sight within the coordinator function. The same goes for errors. By returning a container, such as a Failable, from any operation that can fail (such as I/O), we have to explicitly check if it worked and extract the payload from it only in the success case.

## Logging and monitoring are impure

Logging and monitoring calls are stateful and should be called from your coordinator function just like any other I/O. Don't litter your code with logging statements. If you want to log something in one of your transforms or I/O, instead return any context needed in your return value and then log in your coordinator.

You can see an example of this in our previous getTransactions example. In `db.query` we are construction a failure. Instead of also logging inside of db.query, we return the error wrapped in a failable and log it in our coordinator.

Let's look at another example. It is like our previous example, but we are going to provide a default value for foo in our DTO builder. We are wondering how often this happens so we are going to log each time it does.

```
function getTransactions(accountId) {
    // ...
    const dtos = rows.map(buildTransactionDtoFromDbRow);
    // ...
}

function buildTransactionDtoFromDbRow(row) {
    if (!row.foo) {
        logger.info('row.foo was falsy')
    }

    return {
        id: row.id,
        amount: row.amount,
        createdAt: row.created_at,
        foo: row.foo || 'default foo value'
    };
}
```

Let's pull the logging out of `buildTransactionDtoFromDbRow`. 

```
function getTransactions(accountId) {
    // ...
    const fooUndefinedCount = countTransactionRowsWhereFooIsUndefined(rows)
    if (fooUndefinedCount) {
        logger.info('Found ${fooUndefinedCount} transactions where foo is not defined');
    }

    const dtos = rows.map(buildTransactionDtoFromDbRow);
    // ...
}

function findTransactionRowsWithFooUndefined(rows) {
    return rows.find(row => !row.foo);
}

function buildTransactionDtoFromDbRow(row) {
    return {
        id: row.id,
        amount: row.amount,
        createdAt: row.created_at,
        foo: row.foo || 'default foo value'
    };
}
```

## Creatively avoid special handling of edge cases

Recently my team created an API endpoint to allow a user to create an upvote for an article. There was an edge case where the user could try to upvote for the same guide twice, for example if they had two tabs open. In this case, we want to ignore the second upvote (a user can only upvote once), but we don't want to treat it as an error.

Because of the way we had structured our logic, we needed the POST call for creating an upvote to return the id of the upvote. As a result, if inserting the upvote failed to to a violated unique constraint on the user and article, we could assume the upvote already existed and we needed to query for the ID to return instead of returning a newly created ID.

The code for this edge case was at least half of the code for this endpoint. Tim suggested that we consider this a red flag and try to find a way to creatively restructure our logic so that there was no edge case.

For example, we could restructure our code such that returning the ID is unnecessary. In general, don't only rely on coding techniques to keep things simple.


## Creating a runtime

Your coordinator functions should be very repetitive. They should basically be a list of functions. So why not just [pipe](https://ramdajs.com/docs/#pipe) them? Better yet, why don't we create our own special pipe that handles our logging and monitoring for us?

Before heading down this route, consider a few things.

1. Is it worth it? This layer of abstration will mean that a developer must first understand your pipe function well before they can understand your coordaintor functions. If your project is small, this may not be worth it.
2. The pipe function is a little tricky. Make sure everyone on your team is fully on board and understands your pipe implementation deeply. Having the entire team fully understand and feel ownership over all their code is more important than having clever code such as pipe.

With those warnings aside, here's an example pipe implementation. It doesn't do any monitoring yet, but it does handle logging and short circuiting. You can probably imagine how monitoring could be added without much trouble.

```
export const pipe = R.curry(
  (opts: { logError?: Function }, fns: Function[]) => {
    const { logError = () => {} } = opts;

    return async input => {
      let ctx = { input };
      for (let fn of fns) {
        const result = await fn(ctx);
        if (isFailable(result)) {
          if (result.success) {
            ctx = result.payload!;
          } else {
            logError(result.error);
            return result;
          }
        } else {
          ctx = result;
        }
      }
      return success(ctx);
    };
  }
);
```

Pipe assumes that the result of each function will get passed into the next function. Thefore you might make each function accept and return a context object that can accumulate state (depending on your needs). This means that every function has to conform to this, which can be annoying.

To make this situation a bit better, I created this function:

```
export function mapContext(fn, inputProp, outputProp) {
  return async ctx => {
    const result = await fn(inputProp ? ctx[inputProp] : ctx);
    let mappedResult;
    if (isFailable(result)) {
      mappedResult = mapFailable(
        payload => ({ ...ctx, [outputProp]: payload }),
        result
      );
    } else {
      mappedResult = { ...ctx, [outputProp]: result };
    }
    return mappedResult;
  };
}
```

It allows you to take a value from your context object and use it as an argument to your function and map the result of that function back into the context object. This way you can put functions unaware of this context into your pipe.

The last problem I ran into with pipe's handling of failables. It never passes failables through. Instead it unwraps them. If it is a failure, it logs the error and returns immediately. If it is successful, it returns the unwrapped payload. In one case, there was one type of error that I wanted to ignore. In order to handle this, I created a function to wrap a failable in a success. This way the payload will be another raw failable. Then the next function in the pipe can inspect the error and decide whether or not to return another failure.

It is trivial, but here is a utility to wrap the failable for you in this case.

```
export function catchFailable(fn) {
  return async (...args) => {
    const result = await fn(...args);
    return success(result);
  };
}
```

And finally, here is a coordinator function that uses all these utilities.

```
const buildCoordinator = pipe({
  logError: logger.error.bind(logger)
});

export const createUpvote = buildCoordinator([
  mapContext(validateCreateUpvoteInput, "input", "validatedInput"),
  mapContext(buildUpvote, "validatedInput", "upvote"),
  mapContext(buildInsertUpvoteCommand, "upvote", "insertUpvoteCmd"),
  mapContext(catchFailable(db.query), "insertUpvoteCmd", "insertUpvoteFailable"),
  mapContext(
    transformInsertUpvoteFailure,
    "insertUpvoteFailable",
    "upvoteAlreadyExisted"
  ),
  mapContext(
    buildFetchUpvoteByCompositeIdCommandIfUpvoteAlreadyExisted,
    undefined,
    "fetchUpvoteCmd"
  ),
  mapContext(db.query, "fetchUpvoteCmd", "existingUpvoteQueryResult"),
  buildCreateUpvoteResult
]);
```

After writing this, I think it would be best to use wrap only in cases where the function being wrapped is not specific to our coordinator. In cases where the function wrapped is specific to the coordinator, we might as well just make that function aware of the context and avoid the mental overhead caused by the wrapper.

[1] This is a rough estimate from my recollection. I did not take the time to do a code analysis for this post.