---
layout: post
title:  "Puma Syscalls Analysis"
date:   2021-04-17 21:50 +0800
categories: performance
---

I just copied my PR content here, please check out [Defer checking if socket is closed](https://github.com/puma/puma/pull/2602)

Avoid making `getsockopt` system call for every request

### Description

In function `handle_request`, we can check if the socket is closed when rescuing IOError, saving one `getsockopt` system call for every request.

In the current implementation, the socket may be closed by the client after we check the socket status using `closed_socket?` and before finishing writing the data.

Writing to a closed socket will raise `IOError`. I am not sure which of the following `Error` I should use, my commit uses "Socket timeout writing data"

```
raise ConnectionError, "Connection error detected during write"

raise ConnectionError, "Socket timeout writing data"
```

Analysis detail:

1. Run `hello.ru` benchmark for 30 seconds, 
2. While running benchmark, run `sudo perf stat -e 'syscalls:*' -p $(pidof bundle) sleep 10`, this will use perf to collect all syscalls made by puma process for 10 seconds

perf stat result:

```
 Performance counter stats for process id '1732':

             42009      syscalls:sys_enter_getpeername
             42009      syscalls:sys_exit_getpeername
             42009      syscalls:sys_enter_recvfrom
             42010      syscalls:sys_exit_recvfrom
             84019      syscalls:sys_enter_setsockopt
             84021      syscalls:sys_exit_setsockopt
             42010      syscalls:sys_enter_getsockopt
             42010      syscalls:sys_exit_getsockopt
              8215      syscalls:sys_enter_epoll_ctl
              8215      syscalls:sys_exit_epoll_ctl
              8156      syscalls:sys_enter_epoll_wait
              8156      syscalls:sys_exit_epoll_wait
             37811      syscalls:sys_enter_select
             37811      syscalls:sys_exit_select
                11      syscalls:sys_enter_ppoll
                11      syscalls:sys_exit_ppoll
              6235      syscalls:sys_enter_read
              6235      syscalls:sys_exit_read
             88227      syscalls:sys_enter_write
             88226      syscalls:sys_exit_write
                 1      syscalls:sys_enter_madvise
                 1      syscalls:sys_exit_madvise
                21      syscalls:sys_enter_mprotect
                21      syscalls:sys_exit_mprotect
            274663      syscalls:sys_enter_futex
            274659      syscalls:sys_exit_futex
                 1      syscalls:sys_enter_sched_yield
                 1      syscalls:sys_exit_sched_yield
```

Benchmark RPS is around 4k to 5k, here is my basic understanding of what each syscall is used for:

- `getpeername`, for getting the peer IP address
- `recvfrom`, ?
- `setsockopt`, `cork_socket` and `uncork_socket`, this will be removed in https://github.com/puma/puma/pull/2595
-  `getsockopt`, check socket status to see if the socket is closed
- `epoll_*`, epoll functions used in `nio4r`
- `select`, `IO.select` will get the socket that is ready for reading or writing
- `ppoll`, probably some blocking IO functions
- `read` and `write`, for reading request and writing response data
- `madvise` and `mprotect`, GC stuffs
- `futex`, puma uses threads, `mutex` and `conditional variable`  would cause lot's of `futex` usage, looking forward to getting rid of them(ALL) in https://github.com/puma/puma/pull/2601
- `sched_yield`, process management, https://linux.die.net/man/2/sched_yield



### Your checklist for this pull request
<!--- Go over all the following points, and put an `x` in all the boxes that apply. -->
<!--- If you're unsure about any of these, don't hesitate to ask. We're here to help! -->
- [x] I have reviewed the [guidelines for contributing](../blob/master/CONTRIBUTING.md) to this repository.
- [ ] I have added (or updated) appropriate tests if this PR fixes a bug or adds a feature.
- [x] My pull request is 100 lines added/removed or less so that it can be easily reviewed.
- [ ] If this PR doesn't need tests (docs change), I added `[ci skip]` to the title of the PR.
- [ ] If this closes any issues, I have added "Closes `#issue`" to the PR description or my commit messages.
- [ ] I have updated the documentation accordingly.
- [x] All new and existing tests passed, including Rubocop.
