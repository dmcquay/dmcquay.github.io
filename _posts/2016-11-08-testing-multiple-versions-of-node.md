---
layout: post
title: "Testing Multiple Versions of Node"
date: 2016-11-08
categories: nodejs
---

# SAAS CI

Travis CI and many others support this out of box. Most common to do this and probably recommended in most cases. However, if you need to do it locally for some reason, here are some local scripts.

# Local Script

Really easy to setup.
Don't have to distribute new CI credentials.
Easiest way to provide DB, etc.
Slow to execute.
Doesn't ensure tests get executed.

{% highlight shell %}
#!/bin/bash

# load nvm
. ~/.nvm/nvm.sh

versions=$(cat .node_versions)
command=$@

for version in $versions
do
        nvm install $version
        $command
        if [[ "$?" != "0" ]]
        then
                echo "Failed running $command on $version"
                exit 1
        fi
done

echo "All tests passed"
{% endhighlight %}

# Parallelized Local Script

Faster.
Not always safe.
Solve with Docker? Complicated.

{% highlight shell %}
#!/bin/bash

# load nvm
. ~/.nvm/nvm.sh

versions=$(cat .node_versions)
command=$@

for version in $versions
do
        nvm install $version
        $command &
done
wait

if [[ "$?" == "0" ]]
then
        echo "All tests passed"
        exit 0
else
        echo "One or more tests failed"
        exit $?
fi
{% endhighlight %}
