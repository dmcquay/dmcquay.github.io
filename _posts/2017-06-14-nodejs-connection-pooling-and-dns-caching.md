The following potential concerns came up in my workplace recently, so I decided to investigate them.


- Node.js does not cache DNS responses
- Node.js does not maintain connections to servers


# DNS Caching

http://www.madhur.co.in/blog/2016/05/28/nodejs-dns-cache.html
dig facebook.com only takes 31 ms for me, but it does take < 1 ms subsequent times
30 extra ms per request IS significant for our prod apps
npm module "dnscache"
what i care about is resolving internal.pluralsight.com
dig internal.pluralsight.com from one of our production hosts reported 0 ms!

{% highlight javascript %}
const axios = require('axios')
let promise = Promise.resolve()
for (let i = 1; i <= 20; i++) {
        promise = promise
                .then(() => console.time(`request ${i}`))
                .then(() => axios.get('http://internal.pluralsight.com/learner/content/health-check'))
                .then(() => console.timeEnd(`request ${i}`))
}
{% endhighlight %}

From my laptop:

- request 1: 120.827ms
- request 2: 107.699ms
- request 3: 99.480ms
- request 4: 100.942ms
- request 5: 103.330ms
- request 6: 162.918ms
- request 7: 115.165ms
- request 8: 101.771ms
- request 9: 100.638ms
- request 10: 99.298ms
- request 11: 102.911ms
- request 12: 102.551ms
- request 13: 103.913ms
- request 14: 103.288ms
- request 15: 102.118ms
- request 16: 168.192ms
- request 17: 100.394ms
- request 18: 99.501ms
- request 19: 103.233ms
- request 20: 101.774ms

From a production server:

```
request 1: 27.822ms
request 2: 11.794ms
request 3: 3.940ms
request 4: 8.294ms
request 5: 4.611ms
request 6: 5.022ms
request 7: 6.291ms
request 8: 6.584ms
request 9: 4.967ms
request 10: 4.327ms
request 11: 3.638ms
request 12: 5.538ms
request 13: 3.812ms
request 14: 7.663ms
request 15: 7.919ms
request 16: 3.664ms
request 17: 3.780ms
request 18: 7.269ms
request 19: 13.037ms
request 20: 7.429ms
```

After introducing a 1 second delay between requests (to expire TTL if present):

{% highlight javascript %}
const axios = require('axios')
let promise = Promise.resolve()
for (let i = 1; i <= 20; i++) {
        promise = promise
                .then(() => new Promise(resolve => setTimeout(resolve, 1000)))
                .then(() => console.time(`request ${i}`))
                .then(() => axios.get('http://internal.pluralsight.com/learner/content/health-check'))
                .then(() => console.timeEnd(`request ${i}`))
}
{% endhighlight %}

request 1: 25.890ms
request 2: 12.366ms
request 3: 5.543ms
request 4: 5.251ms
request 5: 7.563ms
request 6: 5.311ms
request 7: 5.762ms
request 8: 6.495ms
request 9: 6.077ms
request 10: 6.665ms
request 11: 5.031ms
request 12: 6.225ms
request 13: 8.152ms
request 14: 7.691ms
request 15: 10.377ms
request 16: 4.827ms
request 17: 17.804ms
request 18: 7.379ms
request 19: 15.028ms
request 20: 9.069ms

Let's try hitting an IP directly:

{% highlight javascript %}
const axios = require('axios')
let promise = Promise.resolve()
for (let i = 1; i <= 20; i++) {
        promise = promise
                .then(() => console.time(`request ${i}`))
                .then(() => axios.get('http://10.107.148.199/learner/content/health-check'))
                .then(() => console.timeEnd(`request ${i}`))
}
{% endhighlight %}



# Request Connection Pooling

https://stackoverflow.com/questions/17375021/how-to-manage-node-js-request-connection-pool
https://nodejs.org/api/http.html#http_class_http_agent

Looks to me like node DOES support connection pooling. But by default, only if there are pending requests to that host. You have to set keepAlive: true in a custom agent to keep it open indefinitely.



# Conclusion

I have not finished the full investigation, but it appears that if there is a remaining issue to be solved,
it will be a very small win. The black box testing above where I was just testing total request
times showed that the current overhead is very small. At least in the case of my application, if I could
shave off another 5ms, it would not be meaningful.
