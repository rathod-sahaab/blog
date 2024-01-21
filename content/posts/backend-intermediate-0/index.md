---
author: "Abhay"
title: "Consitent Hashing: Distribution system of distributed systems."
description: "Whether it's multiple servers, queue consumers, shards of DB or other processes, reliable & stable workload/data distribution is a challenge. A challenge fit for consistent hashing to solve, let's use rust by the way."
date: "2024-01-21"
template: "post"
draft: true
slug: "backend-intermediate-1"
series:
  - "backend-intermediate"
tags:
  - "backend"
  - "rust"
  - "low-level-design"
hidemeta: false
comments: false
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
    image: "posts/backend-intermediate-1/thumbnail-bi1.png" # image path/url
    alt: "Thumbnail: Distribution system of distributed systems" # alt text
    caption: "Consistent hashing in a sentence." # display caption under cover
    relative: true # when using page bundles set this to true
    hidden: false # only hide on current single page
---

## Challenge of distribution in distributed systems
When you have multiple servers handling different request the challenge what server should a request be directed to. Surely, we should not direct all request to one server that defeats the purpose of having multiple servers.

_NOTE: here the term servers is used loosely can mean anything from an API server to a Database._

One thing we can do is store count of all the request a server has been assigned and then assign request to server with lowest request handled. Though this approach works it has some pitfalls.
1. Can only be used for non-persistent workloads.
2. It's stateful procedure, when the server responsible for distribution goes down all data is lost. Reliable state management will cost performance.
3. Dynamically adding and removing servers is difficult.

Another solution is hashing whenever a request comes we hash it to get server details entry stored in a hash table or array. Drawback of this approach are:
1. Dynamically adding and removing servers, require resizing and rehashing of existing data.

## Solution: Consistent hashing
Our expectations from a solution are:
1. Stable distribution
    1. For persistent request types, has to the same values.
    2. Even distribution of requests.
2. Stateless, so even our distribution server goes down we can recover and even scale distribution servers as scale increase.
3. Support Dynamically adding and removing consumers.

## Implementation: Consistent Hash-ring
In our typical with rust approach let's get the contract out of the way.

```rust
pub trait ConsitentHashRing {
    type ConsumerInfo;

    fn add_consumer(&mut self, key: &str, data: Self::ConsumerInfo);
    fn remove_consumer(&mut self, key: &str);

    fn get_consumer(&self, key: &str) -> Option<&Self::ConsumerInfo>;
}
```
Our trait/contract provides 3 functions
1. `add_consumer`
2. `remove_consumer`
3. `get_consumer`: Get consumer details for a non-persistent or persistent request, to which request must be redirected to.

And expects a type `ConsumerInfo` to store the server location data, stored in the utility implementing this contract. This is because contract doesn't know what data you are going to store but it's only logical to return what we were asked stored, this is to enforce just that.

`get_consumer` returns an `Option` for the case when get_consumer is called before any consumer is added. The caller can handle this at their own accord.


