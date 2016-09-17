---
layout: post
title: "Clean Up Git Branches on a Remote"
date: 2016-09-17
categories: git
---

**Merged Branches - These should all be safe to delete**

{% highlight bash %}
for branch in `git branch -r --merged | grep -v HEAD`; do echo -e `git show --format="%cr|%an" $branch | head -n 1`'|'$branch; done | sort -r | column -s '|' -t
{% endhighlight %}

**Unmerged Branches - These need to be looked at closely**

{% highlight bash %}
for branch in `git branch -r --no-merged | grep -v HEAD`; do echo -e $branch'|'`git show --format="%cr|%an|%s" $branch | head -n 1`; done | sort -r | column -s '|' -t
{% endhighlight %}
