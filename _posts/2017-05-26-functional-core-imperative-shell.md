Listed in order of preference. Pure functions preferred over shells which are preferred over "other" functions.

- pure functions:
  - no side effects
  - just take data in and data out
  - they should always produce the same output for the same input (not dependent on result of a DB query, etc)
  - no mocks should ever be needed
  - test using unit tests
- imperative shell
  - must only have one path of execution (no branching)
  - not as critical, but really shouldn't have any logic
  - the top level shell needs to decide what to do with errors, but generally it
    should do so by passing the final promise chain to another function (a pure
    function)
- other functions
  - must be either integration tested or unit tested with mocks
  - must only be called by an imperative shell or "other function"

{% highlight javascript %}
// route handler
// imperative shell
function handleGetCommentsForUserRequest(req, res, next) {
  const userId = req.params.userId
  const promise = getCommentsForUserFacade(userId)
  return sendGetCommentsForUserResponse(res, next, promise)
}

// top level facade
// imperative shell
function getCommentsForUserFacade(userId) {
  const promise = getCommentsForUserFromCache(userId)
    .then(healCommentsForUserFromApi)
  return handle
}

// function that breaks all the rules
// must be integration tested (or unit tested with mocks)
// must be called by an imperative shell
function getCommentsForUserFromCache(userId) {
  return pgClient.execute(...)
}

// function that breaks all the rules
// must be integration tested (or unit tested with mocks)
// must be called by an imperative shell
// TODO: can i make this an imperative shell so i don't have to test it w/ mocks?
const healCommentsForUserFromApi = userId => commentsFromCache => {
  let dtos
  if (commentsFromCache) return Promise.Resolve(commentsFromCache)
  return axios.get('http://example.com/some/path/' + userId)
    .then(buildUserCommentsDtos)
    .then(r => dtos = r)
    .then(saveUserCommentDtos)
    .then(() => dtos)
}

// function that breaks all the rules
// must be integration tested (or unit tested with mocks)
// must be called by an imperative shell
function saveUserCommentDtos(dtos) {
  return dtos.map(dto => pgClient.execute('insert into ...', Object.values(dto)))
}

// pure function
// unit tested
function buildUserCommentsDtos(data) {
  return new Dto({
    foo: data.foo,
    ...
  })
}

function getCommentsForUserFromApi(userId) {

}
{% endhighlight %}
