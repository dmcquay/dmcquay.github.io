# How to stream DB results

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

## Result

- Low memory footprint
- Slower!

What? Slower? Yes, by a lot. I was using compress middleware for express and I noticed that streaming the response data prevented that middleware from working. My time to last byte for local requests when from ~3s to ~10s. I enabled compression by manually piping through a zlib stream like this:

```
const zlib = require('zlib')
const z = zlib.createGzip()
res.set('Content-Encoding', 'gzip')
streamOfObjects.pipe(JSONStream.stringify()).pipe(z).pipe(res)
```

This made the response once again gzipped and brought the time to last byte down to ~4.2s. Must better, but still slower than the original.
