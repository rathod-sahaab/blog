---
title: Accelerating libnbd on linux using io_uring [1]
date: '2021-06-01T19:12+00:00'
template: 'post'
draft: false
slug: 'libnbd-io_uring-1'
category: 'Systems Programming'
tags:
  - 'libnbd'
  - 'iouring'
  - 'systems-programming'
description: 'io_uring is the new and better way to do asynchronus I/O on linux, tag along with me on a journey to learning of doing on a real life project libnbd!'
socialImage: '/media/io_uring-sq-cq.png'
---

# Research Notes

This post contains some research notes so don't expect great quality, this is just to document how I go around devising a solution so you don't assume me as some systems programming wizard.

I will merge my notes from earlier in first note and will add dates to later notes.

### Example projects

- [wcp](https://github.com/wheybags/wcp)

## 2nd June 2021

### Concepts

#### libnbd related:

2 copies are done during a nbdcopy command one when kernel transfers data to userspace and other when userspace process (nbdcopy) send to the kernel to send it to destination. io_uring can help us avoid the second copy which is significant when we look at the _flamegraph_.

#### io_uring related:

[Submission Queue Polling](https://unixism.net/loti/tutorial/sq_poll.html): A nice feature where kernel threads polls _SQEs_ for you so you don't have to call `io_uring_enter(2)`, set the flag `IOURING_SETUP_SQEPOLL` in `io_uring_params` when

[Linked Requests](https://unixism.net/loti/tutorial/link_liburing.html#linking-requests): Submission are not processed in order they were submitted, their completion time depends on many different factors but it's better to assume they are completed in random order. Linked Requests enable us to ensure some requests only start after other are completed.

### Code

Using io_uring directly is requires complex boilerplate which though not too difficult but is wasted effort. For this purpose, liburing helps code code is smaller and hence easier to maintain.

#### liburing

##### setup

```c
#define ENTRIES 32

struct io_uring ring;
io_uring_queue_init(ENTRIES, &ring, 0);
```

##### submission

```c
struct io_uring_sqe sqe;
struct io_uring_cqe cqe;
/* get an sqe and fill in a READV operation */
sqe = io_uring_get_sqe(&ring);
io_uring_prep_readv(sqe, fd, &iovec, 1, offset);
/* tell the kernel we have an sqe ready for consumption */
io_uring_submit(&ring);
/* wait for the sqe to complete */
io_uring_wait_cqe(&ring, &cqe);
/* read and process cqe event */
app_handle_cqe(cqe);
io_uring_cqe_seen(&ring, cqe);
```

#### iovec

```c
char *str0 = "hello ";
char *str1 = "world\n";
struct iovec iov[2];
ssize_t nwritten;

iov[0].iov_base = str0;
iov[0].iov_len = strlen(str0);
iov[1].iov_base = str1;
iov[1].iov_len = strlen(str1);

nwritten = writev(STDOUT_FILENO, iov, 2);
```

#### sysconf
how to determine cache size
```c
long cache_size = _SC_LEVEL2_CACHE_SIZE;
```

#### Sample liburing program (write to file)
```c
#include <liburing.h>

#include <fcntl.h>
#include <string.h>
#include <unistd.h>
#define QD 32

int main() {
  struct io_uring ring;
  io_uring_queue_init(QD, &ring, 0);

  struct io_uring_sqe *sqe;
  sqe = io_uring_get_sqe(&ring);

  char buff[3][512] = {"Hello ", "again ", "world!\n"};

  int fd = creat("myfile.txt", O_TRUNC);
  struct iovec iov[3];
  for (int i = 0; i < 3; ++i) {
    iov[i].iov_base = buff[i];
    iov[i].iov_len = strlen(buff[i]) * sizeof(char);
  }
  io_uring_prep_writev(sqe, fd, iov, 3, 0);
  io_uring_submit(&ring);

  struct io_uring_cqe *cqe;
  // wait for events to complete
  io_uring_wait_cqe(&ring, &cqe);
  // mark event as reaped
  io_uring_cqe_seen(&ring, cqe);
  // exit
  io_uring_queue_exit(&ring);

  close(fd);

  return 0;
}
```
