---
title: checking shell scripts on travis
tags: bash travis testing shellcheck
---

Whenever I write shell scripts (usually in Bash) I find it a good practice to
run `shellcheck` to make sure that I haven't done any obvious mistakes, such as
forgetting to quote variables (which happens every now and then).

When I was tinkering with [ddns-route53]({% post_url 2016-01-05-ddns-route53 %})
I thought that it would be nice if Travis could run `shellcheck` for me.
I had some trouble with getting `shellcheck` installed, but eventually found a
solution after hopping through a bunch of GitHub issues (don't even know where I
ended up).

The following `.travis.yml` should get you up and running in no time:

```yaml
language: bash
sudo: required
dist: trusty
before_install:
  - echo "deb http://archive.ubuntu.com/ubuntu/ trusty-backports universe" | sudo tee -a /etc/apt/sources.list
  - sudo apt-get update -qq
  - sudo apt-get install shellcheck -y
script:
  - shellcheck $FILENAME
```

`$FILENAME` could also be replaced with a glob or something like `$(find . -maxdepth 1
-type f -executable)` if you have several files that needs to be checked.

Speaking of shell scripts, I sometimes find myself returning to Thoughtbot's
[The Unix Shell's Humble If](https://robots.thoughtbot.com/the-unix-shells-humble-if)
blog post when I forget the how's and why's of constructing an `if` statement in
Bash.
