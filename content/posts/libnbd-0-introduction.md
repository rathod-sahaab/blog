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
description: 'io_uring is the new and better way to do asynchronus I/O on linux, tag along with me on a journey to learn it by doing, on a real life project libnbd!'
socialImage: '/media/io_uring-sq-cq.png'
---

## What is io_uring?

Though there are already many ways to do async I/O on linux but io_uring the new feauture is way faster than other because of following reasons:

- Shared data between kernel and process avoiding copying large amount of data on every syscall.
- Lock free shared data between kernel and process hence faster.
- Ability to batch up multiple I/O requests (for different files) in one _syscall_ decreasing total number of syscalls required.

[Official Paper on io_uring](https://kernel.dk/io_uring.pdf) by the author of io_uring Jens Axboe goes in much more details of how and why, so make sure to give this masterpiece a read. The best practice it to **NOT** use io_uring but [liburing](https://github.com/axboe/liburing) a library made to simplify using io_uring.

## Motivation

`nbdcopy` one of the most prominent users of libnbd can be and is used by many applications like `virt-v2v`, `oVirt`, `KubeVirt`, etc.

`virt-v2v` tool converts virtual machines (VMs) from foreign hypervisors, including their disk images and metadata, for use with with KVM managed by libvirt.

Using io_uring would improve efficiency and speed hence better performance for programs that use `libnbd` and `nbdcopy`.

Generally, **VMware ESXi** VM images are converted to **KVM** images, hence, the faster we do it the faster we get more opensource :) While copying the images, VMs cannot be operational i.e. downtime until copy is complete and we all know how bad it is... the faster we get out the better it is.

## libnbd

NBD — Network Block Device — is an Application layer protocol for accessing Block Devices over network,
(hard disks and disk-like things) over a Network.

`liburing` is a NBD client library in userspace that lets you perform NBD operations.
The library is hosted at [libguestfs/libnbd](https://github.com/libguestfs/libnbd).

## Future posts

I will go in details as I go forward with the project via new posts. But that's it for now. Thanks for tuning in.

---

## Background

I was learning about operating systems and got interested in systems programming, I was casually browsing through GSoC 2020 ideas to look for an interesting one (which is a pretty good way to find projects). I was immediately taken by the _libnbd / nbdcopy accleration using io_\__uring_ on [libvirt's GSoC page](https://wiki.libvirt.org/page/Google_Summer_of_Code_Ideas). I immediately pinged Richard about if the project was up for taking, it was. So, I started learning and planning for the project and Thanks to Richard's guidance the process was much easier.

As my college requires some kind of internship after 6th sem, I applied for GSoC and my proposal was selected for Google Summer of Code. Officially, coding begins on 7 June 2021 but I have began testing waters for this daunting task already.
