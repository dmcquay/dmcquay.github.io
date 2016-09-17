---
layout: post
title: "Migrate DNS Zone From Rackspace to AWS"
date: 2015-11-20 08:21:00
categories: rackspace
---

First authenticate
http://docs.rackspace.com/servers/api/v2/cs-gettingstarted/content/curl_auth.html

{% highlight bash %}
curl -s https://identity.api.rackspacecloud.com/v2.0/tokens -X 'POST' \
       -d '{"auth":{"RAX-KSKEY:apiKeyCredentials":{"username":"[your-username]","apiKey":"[your-api-key]"}}}' \
       -H "Content-Type: application/json"
{% endhighlight %}

Then:
https://community.rackspace.com/products/f/25/t/1743

{% highlight bash %}
CURL -X GET https://lon.dns.api.rackspacecloud.com/v1.0/ACCOUNTID/domains/DOMAINID/export -H "X-Auth-Token: TOKEN" -H "Content-Type: application/xml" -H "Content-Length: 0"
{% endhighlight %}

grab the callback URL:
{% highlight bash %}
CURL -X GET [callbackUrl]?showDetails=true -H "X-Auth-Token: TOKEN" -H "Content-Type: application/xml" -H "Content-Length: 0"
{% endhighlight %}

wait for it to say status is "COMPLETED"
and content type is BIND_9

copy the contents, including quotes
open chrome browser, dev tools
c = [paste]
document.body.innerHTML = '<pre>' + c + '</pre>';
then copy the contents of the page

now open Route 53, create the new zone, click "Import Zone File"
paste this content, click "Import", refresh to see results
