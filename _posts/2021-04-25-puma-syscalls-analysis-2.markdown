---
layout: post
title:  "Puma Syscalls Analysis 2"
date:   2021-04-25 00:05 +0800
categories: performance
---

This is following work to my last [blog post](/performance/2021/04/17/puma-syscalls-analysis.html)

Notice that puma is making `getpeername` and `recvfrom` in every request.

```
42009      syscalls:sys_enter_getpeername
42009      syscalls:sys_exit_getpeername
42009      syscalls:sys_enter_recvfrom
42010      syscalls:sys_exit_recvfrom
```

Because, puma will [reset](https://github.com/puma/puma/blob/b9a121be8f878a1fa7dc71afabc383f5957cbf49/lib/puma/client.rb#L129) `Client` instance `@peerip` after every request. So a new request will use the `@io.peeraddr` method to [get the client ip](https://github.com/puma/puma/blob/b9a121be8f878a1fa7dc71afabc383f5957cbf49/lib/puma/client.rb#L234).

After a TCP connection is established, client IP won't change, if client changes IP, it will initiate a new TCP connection.

I don't think MPTCP is popular yet, so this situation is not considered.

A [PR](https://github.com/puma/puma/pull/2609) is approved and waiting to be merged. By saving two syscalls on every request, `hello.ru` benchmark shows 27% throughput improvement.