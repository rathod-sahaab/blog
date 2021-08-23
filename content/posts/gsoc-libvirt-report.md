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

This summer I participated in <abbr title="Google Summer of Code">GSoC</abbr> for the project **Accelerating libnbd on linux using io_uring**. Here is a summary of roughly 200hrs of work. This is just an high level overview to mark the end you can find links to detailed posts at the end.

## nbdcpy

_The proof of concept application_ to test features of `io_uring` and optimizing approach before integrating into libnbd + nbdcopy.

### Rationale

Early on during the project it became clear that making libnbd use `io_uring` was too big of a task for me as I had to manage both libnbd and `io_uring` together. Libnbd is very simple to use but that is only made possible by a complex machinery underneath, the complexity is inevitable when you consider the design of the NBD protocol which it processes. NBD is a **stateful**, **binary protocol** which supports **out of order multiplexing** to share binary data over network from a block-like device. <b>nbdcopy</b> (not nbdcpy) can support multiple sources ranging from actual **block devices** and **nbd servers** to **pipes** and **files**.

As playing with the real code will make it a bit difficult to not only develop and debug the feature but also trying radical approaches will be more pain.

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

I chose C++ because I am more comfortable with it, but code can easily be translated to C(the goal).

#### Challenges

The major challenges were caused by multiplexing feature of NBD protocol some of most prominent ones.

##### Completion event for reads

The way we use `io_uring` is by submitting events and waiting for some event to complete, on completion we get a cqe which marks that event is completed and we can move to next stage.

```mermaid
     ┌─────────────────────────────┐
     │  Read Request Sent(source)  ◄────────┐
     └─────────────┬───────────────┘        │
                   │                        │
                   │                        │
                   │ CQE (completion)       │
                   │                        │
                   │                        │
    ┌──────────────▼──────────────┐         │
    │ Reading From Socket(source) │         │
    └──────────────┬──────────────┘         │
                   │                        │
                   │                        │
                   │  CQE (completion)      │ CQE
                   │                        │
                   │                        │
  ┌────────────────▼─────────────────┐      │
  │ Sending data to dest(destination)│      │
  └────────────────┬─────────────────┘      │
                   │                        │
                   │                        │
                   │ CQE (completion)       │
                   │                        │
                   │                        │
     ┌─────────────▼──────────────┐         │
     │ Reading Confirmation(dest) ├─────────┘
     └────────────────────────────┘
```

As state earlier NBD protocol supports multiplexing and one should use it to get better performance.

What it means is you send multiple requests before any of your requests are fulfilled.

You send:

```
Request[0] + Request[1] + Request[2] + ...
```

But it's not guaranteed that you'll response in same order, for example, you can receive:

```
Response[1] + Response[2] + Response[0] + ...
```

For writes it causes no problem we can always determine which write we completed but determining that for reads become complex.

To put it simply we can't determine which request was fulfilled, and hence we first need to read packet to determine which request does it belongs to.

Reading whole packet is also tricky as we will see in next part.

##### Errorneous reads

NBD response can look like below when everything goes right.

```
HEADER1 + DATA1 + HEADER2 + DATA2 + HEADER3 + DATA3 ...
```

and when things go wrong, we can receive following response, that is \*\*no data part accompanies the header.

```
HEADER1 + HEADER2 + HEADER3 ...
```

That means we can't just say that our NBD header was 28 bytes and in all our we 512 bytes, so Mr. IOuring please give us 540 bytes. We absolutely NEED to read header first check for errors and then only can read for data. We can only read one nbd packet after previous packet was read due to property of TCP sockets.

Solution was to only queue one Header read at a time in `io_uring` submission queue at a time and process it then queue another read(this is not very efficient but works).

## Alternative approach

Another approach which is on my radar is when socket is available to read we read all of it, and then extract packets from read data accordingly in user space. What this allows us to do is process multiple packets in single read and will be able to batch request together to better utilize `io_uring` features.

#### source & documentation

[gitlab.com/rathod-sahaab/nbdcpy](https://gitlab.com/rathod-sahaab/nbdcpy)

## Active Bugs

1. When copying data more than 3 MB, program mostly stops after a random point. Which might be caused due to non-existent handling of NBD errors or buggy code.

## Upnext

`io_uring` is rapidly evolving and many new features are being added that might be helpful in optimizing the code even further.
After bugs are fixed next point on the agenda is to

1. Clean up code and update documentation then ask for reviews from people more involved & experienced
   with `io_uring` regarding how can this be made better.
2. Profiling different approaches, and measuring performance <abbr title="with respect to">wrt</abbr> different variables.
3. Engineering and integrating a solution into libnbd.
