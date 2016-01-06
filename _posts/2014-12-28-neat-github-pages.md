---
title: neat github pages with bourbon and bitters
tags: github bourbon neat bitters
---

Several times during this past year I've thought to myself that maybe I should
write a little something about problems that I encounter and solve when fiddling
with things, or tools that I find useful, but never really got to it.

During the Christmas downtime I decided to make an attempt to create a small
blog using Github Pages, as it seemed rather straight forward to get something
up 'n running.
I also wanted to try my luck (again) with the `bourbon`, `neat`, and `bitters`
gems from thoughtbot.

Assuming you've got the `github-pages` gem installed and the basics already set
up, the first step is to add the gems to your `Gemfile`, or just install them
with `gem` if you're not using `Bundler`.

```ruby
gem 'bitters'
gem 'neat'
gem 'bourbon'
```

Since GitHub runs `jekyll` with the `--safe` flag we'll need to install the
stylesheets locally, which is done by running the following commands in the
configured `sass_dir` (by default `_sass`):

```shell
$ bourbon install
$ neat install
$ bitters install
```

The last piece of the puzzle is to add some `@import` statements to our main
SASS stylesheet, for example:

```sass
@import "bourbon/bourbon";
@import "base/base"; // bitters needs to be imported before neat
@import "neat/neat";
```

That's about it, now you should have a `jekyll` blog set up with `bourbon`,
`neat` and `bitters`, ready to be customized to your liking.
I'd start with looking into `base/variables.scss` and `base/grid-settings.scss`.

Given the "design" of this blog, using a grid framework and whatnot might seem a
bit unnecessary, but I like experimenting with new toys and the best way, in my
opinion, is to actually get something done.
