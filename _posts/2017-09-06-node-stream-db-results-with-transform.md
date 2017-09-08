# How to stream DB results

In my current use case, I am using mysql. To get a stream of results, we simply
call `query.stream()`.

I use Postgres more often, so I was curious if my go-to Pg library provided a
stream interface as well. It turns out you need another library for that.

https://github.com/brianc/node-pg-query-stream

```
npm i -S pg pg-query-stream JSONStream
```

```
var pg = require('pg')
var QueryStream = require('pg-query-stream')
var JSONStream = require('JSONStream')

//pipe 1,000,000 rows to stdout without blowing up your memory usage
pg.connect(function(err, client, done) {
  if(err) throw err;
  var query = new QueryStream('SELECT * FROM generate_series(0, $1) num', [1000000])
  var stream = client.query(query)
  //release the client when the stream is finished
  stream.on('end', done)
  stream.pipe(JSONStream.stringify()).pipe(process.stdout)
})
```

## How to transform the objects to VM shape

Likely we will want to change the shape of our data a bit before returning it,
such as building a VM. To do this with a stream, we will need to create our own
Transform steam.

Stream Docs: https://nodejs.org/api/stream.html

```
const {Transform} = require('steam')
const buildVmTransform = new Transform({
  objectMode: true,

  // same as both of these together:
  // readableObjectMode: true,
  // writableObjectMode: true,

  transform(chunk, encoding, callback) {
    this.push(buildVm(chunk))
    callback()
  }
})
```

## Streaming the objects in JSON format

Need a stream that is in buffer mode and outputs in JSON (not terribly natural as a stream format).

JSONStream will take care of all of this for us.

 - Transform steam, which is a type of Duplex stream
 - Readable and writable
 - Readable stream is in object mode
 - Writeable stream is in buffer mode

## Listening to stream events instead of piping

You might consider serializing the data before sending to the response similar
to this:

```
res.write('[')
let first = true
streamOfObjects.on('data', obj => {
    if (first)
        first = false
    else
        res.write(',')
    res.write(JSON.stringify(obj))
})
streamOfObjects.on('end', () => {
    res.write(']')
    res.end()
})
```

This has one major problem. Streams can provide back pressure and this is
critical. The method above does not handle that. You could add support for that
by watching for back pressure from the response writable stream, calling pause
on `streamOfObjects` and then listening to the `drain` event on the response
and resuming on the `streamOfObjects` in that case. But If you simply do your
transform using a proper stream, when you pipe the streams together, the back
pressure handling is done for you automatically.

## JSONStream

We need a stream that is in buffer mode and outputs in JSON (not terribly
natural as a stream format).

JSONStream will take care of all of this for us.

 - Transform steam, which is a type of Duplex stream
 - Readable and writable
 - Readable stream is in object mode
 - Writeable stream is in buffer mode

## Result

- Low memory footprint
- Slower!

What? Slower? Yes, by a lot.

## Compression

I was using compress middleware for express and I noticed that streaming the response data prevented that middleware from working. My time to last byte for local requests when from ~3s to ~10s. I enabled compression by manually piping through a zlib stream like this:

```
const zlib = require('zlib')
const z = zlib.createGzip()
res.set('Content-Encoding', 'gzip')
streamOfObjects.pipe(JSONStream.stringify()).pipe(z).pipe(res)
```

This made the response once again gzipped and brought the time to last byte down to ~4.2s. Much better, but still slower than the original ~3s.

## Effect of streaming results from DB

Next I wanted to rule out if the streaming interface to the DB was slow, so
I wrote two scripts to compare. Simultaneously I am testing if it is slower to
serialize many individual objects than it is to serialize one large array of
objects all at once.

This script tests fetching all objects at once and serializing them all at once.
On my laptop, it completes in ~3.5s.

```
const repo = require('./claims/repo')
const mysql = require('./mysql')

console.time('getClaims')
repo.getClaims()
    .then(claims => {
        mysql.close()
        JSON.stringify(claims)
        console.timeEnd('getClaims')
    })
```

This next script fetches the objects via a stream and serializes them one at a
time, never storing more than one at a time in memory. It completes in ~2.9s.

```
const repo = require('./claims/repo')
const mysql = require('./mysql')

console.time('getClaimsStream')
const claimsStream = repo.getClaimsStreaming()
claimsStream.on('data', claim => JSON.stringify(claim))
claimsStream.on('end', () => {
    mysql.close()
    console.timeEnd('getClaimsStream')
})
```

Therefore we are actually able to read data from the database *faster* when using
the stream interface and serializing objects individually.

We still need to figure out where we are losing about a second.

## JSONStream

Next I decided to test if `JSONStream` is slow. So I wrote my own Transform stream
that serializes each object using `JSON.serialize`. For each test I loaded all
my claims and then serialized them all to make it valid against my specific
payload.

Here is the script to test JSONStream. It executes in ~2.9s.

```
console.time('JSONStream')
const JSONStream = require('JSONStream')
const testStream = repo.getClaimsStreaming().pipe(JSONStream.stringify())
testStream.on('data', () => {})
testStream.on('end', () => {
    mysql.close()
    console.timeEnd('JSONStream')
})
```

Here is the script that compares it to a custom stream that only relies on
`JSON.stringify`, which I already proved performs well. It executes in ~3.2s.

```
console.time('CustomJSONTransformStream')
const {Transform} = require('stream')
const serialize = new Transform({
    writableObjectMode: true,
    transform(chunk, encoding, callback) {
        this.push(JSON.stringify(chunk))
        callback()
    }
})
const testStream = repo.getClaimsStreaming().pipe(serialize)
testStream.on('data', () => {})
testStream.on('end', () => {
    mysql.close()
    console.timeEnd('CustomJSONTransformStream')
})
```

So apparently JSONStream is efficient and does not seem to be our problem.

## High Water Mark

I found that JSONStream does not allow me to configure a highWaterMark setting
and that the default value of 16 (for object mode) and 16k (for buffer mode)
was sub-optimal for my use case.

I set the highWaterMark for my DB stream like this: `query.stream({highWaterMark: 500})`

To set the highWaterMark for my JSON serialization stream, I had to create my
own since JSONStream doesn't allow that to be configured. Here's what my
implementation looks like:

```
let first = true
const serialize = new Transform({
    writableObjectMode: true,
    highWaterMark: 500,
    transform(chunk, encoding, callback) {
        if (first) {
            this.push('[' + JSON.stringify(chunk))
            first = false
        } else {
            this.push(',' + JSON.stringify(chunk))
        }
        callback()
    },
    flush(callback) {
        this.push(']')
        callback()
    }
})
```

Finally, I increased the highWaterMark value for the gzip stream (which
operates in buffer mode) to ten times the default like this:
`zlib.createGzip({highWaterMark: 16384 * 10})`.

I played with these values quite a bit to find the optimal combination and it
improved my response time to ~3.8s.

## Going forward

Sadly, I could never achieve the performance of my original, non-stream
implementation. I still may use the stream approach because without it I have a
significant memory problem. If you have any ideas why the stream approach is
still a bit slower, please let me know!
