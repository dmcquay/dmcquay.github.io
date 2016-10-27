---
layout: post
title: "Clean Up Git Branches (Remote & Local)"
date: 2016-09-17
categories: git
---

# Cleaning up a remote

**Merged Branches - These should all be safe to delete**

{% highlight bash %}
for branch in `git branch -r --merged | grep -v HEAD`; do echo -e `git show --format="%cr|%an" $branch | head -n 1`'|'$branch; done | sort -r | column -s '|' -t | grep -v master
{% endhighlight %}

**Unmerged Branches - These need to be looked at closely**

{% highlight bash %}
for branch in `git branch -r --no-merged | grep -v HEAD`; do echo -e $branch'|'`git show --format="%cr|%an|%s" $branch | head -n 1`; done | sort -r | column -s '|' -t
{% endhighlight %}

# Cleaning up local branches

**Merged Branches - These should all be safe to delete**

{% highlight bash %}
git branch --merged master | grep -v "\* master"
{% endhighlight %}

If you want to delete them automatically:

{% highlight bash %}
git branch --merged master | grep -v "\* master" | xargs git branch -d
{% endhighlight %}
