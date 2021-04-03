---
layout: post
title:  "Tidb 第一个 PR - 替换 binary.Read"
date:   2021-04-03 20:05:00 +0800
categories: performance
---

[commit](https://github.com/calvinxiao/tidb/commit/46d29731b4520f8b49c0c6ab8297c8b2f9018a94)

Cool!!

```
name                     old time/op    new time/op    delta
DecodeEscapedUnicode-16     151ns ± 0%      23ns ± 0%   -85.00%  (p=0.000 n=9+10)

name                     old alloc/op   new alloc/op   delta
DecodeEscapedUnicode-16     56.0B ± 0%      0.0B       -100.00%  (p=0.000 n=10+10)

name                     old allocs/op  new allocs/op  delta
DecodeEscapedUnicode-16      4.00 ± 0%      0.00       -100.00%  (p=0.000 n=10+10)
```