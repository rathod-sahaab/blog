---
title: Accelerating libnbd on linux using io_uring [0]
date: '2021-05-24T06:31+00:00'
template: 'post'
draft: false
slug: 'libnbd-io_uring-0'
category: 'Systems Programming'
tags:
  - 'libnbd'
  - 'iouring'
  - 'systems-programming'
description: 'io_uring is the new and better way to do asynchronus I/O on linux, tag along with me on a journey to learning of doing on a real life project libnbd!'
socialImage: '/media/io_uring-sq-cq.png'
---

## Background

I was learning about operating systems and got interested in systems programming, I was casually browsing through GSoC 2020 projects to look for an interesting one (which is a pretty good way to find projects). I was immediately taken by the _libnbd / nbdcopy accleration using io_\__uring_ on [libvirt's GSoC page](https://wiki.libvirt.org/page/Google_Summer_of_Code_Ideas). I immediately pinged Richard about if the project was up for taking, it was. So, I started learning and planning for the project and Thanks to Richard's guidance the process was much easier.

As my college requires some kind of internship after 6th sem, I applied for GSoC and my proposal was selected for Google Summer of Code. Officially, coding begins on 7 June 2021 but I have began testing waters for this daunting task already.

## Motivation

Using io_uring would improve efficiency and thoroughput hence better performance for programs that us libnbd.

## io_uring

Though there are already many ways to do async I/O on linux but io_uring is faster because of following reasons:

- No locks on the shared data between kernel and process.
- Ability to submit multiple I/O request in one _syscall_.

[Official Paper on io_uring](https://kernel.dk/io_uring.pdf) by Jens Axboe goes in much more details, so make sure to give this masterpiece a read. The best practice it to **NOT** use io_uring but [liburing](https://github.com/axboe/liburing) a library made to simplify using io_uring.

## libnbd

It is a NBD client library in userspace. The library is hosted at [libguestfs/libnbd](https://github.com/libguestfs/libnbd). It enables programmers to read from and right to block-devices programatically.

### Network Block Device

NBD — Network Block Device — is a protocol for accessing Block Devices
(hard disks and disk-like things) over a Network. This is the NBD
client library in userspace, a simple library for writing NBD clients.

Basic idea reading and writing blocks of binary data over the network.

## Future posts
I will go in details as I go forward with the project via new posts. But that's it for now. Thanks for tuning in.
