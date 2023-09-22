---
author: "Abhay"
title: Recipie for Node.js - Node.js Deep Dive [0]
description: ""
date: "2023-02-11T00:00:00+00:00"
template: "post"
draft: true
slug: "node-js-deep-dive-0"
series:
  - "nodejs-deep-dive"
tags:
  - "backend"
  - "nodejs"
draft: true
hidemeta: false
comments: false
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---

# Recipe for Node.js
According to [nodejs.org](https://nodejs.org/en/)
> Node.js® is an open-source, cross-platform, asynchronous event-driven JavaScript runtime, Node.js is designed to build scalable network applications.

But, what if every node repository that exists is deleted. How would we get there again?
First of all, no! because [deno](https://deno.land/).
However, lets prepare for another world ending scenario which is definitely never going to happen
and I heroically save the day by writing *node* from scratch.

## Ingredients
### V8: The Brains
V8 refers to a class of internal combustion engines with 8 cylinders known for their power and reliability,
whose name was conviniently stolen by Google for their
"open source high-performance JavaScript and WebAssembly engine, written in C++".

Jokes aside, [V8](https://v8.dev) is the engine/processor/runner that runs JavaScript for node. But, the thing is V8 was made for the web browsers so it was not paired with utilities that can modify you.


