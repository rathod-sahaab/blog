---
title: GSoC Progress Report
date: '2021-08-18T15:17:01+00:00'
template: 'post'
draft: false
slug: 'gsoc-libvirt-report'
category: 'Systems Programming'
tags:
  - 'libnbd'
  - 'iouring'
  - 'systems-programming'
description: 'Blog post to summarise the progress of my GSoC project improving libnbd using io_uring.'
socialImage: '/media/io_uring-sq-cq.png'
---

## Up to speed

This summer I participated in <abbr title="Google Summer of Code">GSoC</abbr> for the project **Accelerating libnbd on linux using io_uring**. Here is a summary of roughly 200hrs of work.

## nbdcpy

_The proof of concept application_

### Rationale

Early on during the project it became clear that making libnbd use `io_uring` was too big of a task for me as I had to manage both libnbd and `io_uring` together. Libnbd is very simple to use but that is only made possible by a complex machinery underneath, the complexity is inevitable when you consider the design of the NBD protocol which it processes. NBD is a **stateful**, **binary protocol** which supports **out of order multiplexing** and <b>nbdcopy</b> (not nbdcpy) can support multiple sources ranging from actual **block devices** and **nbd servers** to **pipes**. Would make it very difficult to develop and debug the feature.

So, my mentor Mr. Richard suggested to write an application that works similar to nbdcopy but should use `io_uring` to get a hang of how `io_uring` works and how to make it work with the NBD protocol.

### Requirements

To be an accurate representation of what libnbd+nbdcopy do nbdcpy should support following features. (verified field marked).

1. [x] Multi-plexing requests on single TCP connection.
2. [ ] Multi-threading
3. [ ] Multi-connection: Multiple TCP connection to an nbdserver to increase throughput.

From the given feature multiplexing is required by the NBD protocol so ensuring that was the first priority. Other two(multi-threading & multi-connection) are optional and are easy to implement once multiplexing is achieved.

### Implementation

#### Technologies

- C++17
- CMake (build)
- conan (package management)

I chose C++ because I am more comfortable with it, but can easily be translated to C(the goal).

#### Challenges

The major challenge was caused by multiplexing feature of NBD protocol. This caused improper reads.

NBD response can look like below when everything goes right

```
HEADER1 + DATA1 + HEADER2 + DATA2 + HEADER3 + DATA3 ...
```

and when things go wrong, we can receive following response

```
HEADER1 + HEADER2 + HEADER3 ...
```

We NEED to read header first and then only can read for data. We can only read
one nbd packet after previous packet was read due to limitation of TCP sockets.

Solution was to only queue on Header read at a time in `io_uring` submission queue details of which can be found in source code.

#### source & documentation

[gitlab.com/rathod-sahaab/nbdcpy](https://gitlab.com/rathod-sahaab/nbdcpy)

## Active Bugs

1. When copying data more than 3 MB, program mostly stops after a random point. Which might be caused due to improper handling of NBD errors or buggy code.

## Future

`io_uring` is rapidly evolving and many new features are being added that might be helpful in optimizing the code even further.
After bugs are fixed next point on the agenda is to

1. Clean up code and update documentation then ask for reviews from people more involved & experienced
   with `io_uring` regarding how can this be made better.
2. Engineering and integrating a solution into libnbd.
