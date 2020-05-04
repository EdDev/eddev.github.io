---
title: "Hello Jekyll!"
date: 2019-05-27
categories:
  - blog
tags:
  - Jekyll
---

# Overview
This is probably my first blog post.
I thought to warm up by sharing how I met Jekyll and introduced it to my
container friend, Mr. podman.

It all started while preparing my talk to
[PyCon IL](https://pycon.org.il/2019/). I realized that a blog could have
been very helpfull in gathering the ideas for such a talk.
Lesson learned, I am on it.

# Jekyll
After a little search, I lended up with Jekyll, a GitHub friend which is
open source, simple and a nice looking static website/blog generator.

So what do I need to make my blog work?
- Education helps:
  - The offcial [docs](https://jekyllrb.com/docs/).
  - Some [youtube](https://www.youtube.com/playlist?list=PLLAZ4kZ9dFpOPV5C5Ay0pHaa0RJFhcmcB).
  - Knowing about the [GitHub themes](https://pages.github.com/themes/).
  - Getting to know a [cool theme](https://github.com/mmistakes/minimal-mistakes).
- Choose the theme.
- Deploy and [run locally](#my-containarized-jekyll) to review work before
  publishing.
- Write content.

## My containarized Jekyll
Here comes podman for the rescue.

Assuming you have prepared the needed Jekyll configuration and backbone
(see https://github.com/EdDev/eddev.github.io for an example), you are ready
to launch your masterpiece.

To run the Jekyll locally:
- build the [container](../../automation/Dockerfile.jekyll)
```
sudo podman build --rm -t jekyll-server -f ./automation/Dockerfile.jekyll .
```
- enter its console:
```
sudo podman run --rm -ti -v ${PWD}:/jekyll:Z --net=host jekyll-server bash
```
- run the server:
```
cd jekyll
./automation/run-server.sh
```
