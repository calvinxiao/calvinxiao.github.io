---
layout: post
title:  "Tidb 第一个 PR - 替换 binary.Read"
date:   2021-04-03 20:05:00 +0800
categories: performance
---

[https://github.com/pingcap/tidb/pull/23845](https://github.com/pingcap/tidb/pull/23845)

Cool!!

```
name                     old time/op    new time/op    delta
DecodeEscapedUnicode-16     151ns ± 0%      23ns ± 0%   -85.00%  (p=0.000 n=9+10)

name                     old alloc/op   new alloc/op   delta
DecodeEscapedUnicode-16     56.0B ± 0%      0.0B       -100.00%  (p=0.000 n=10+10)

name                     old allocs/op  new allocs/op  delta
DecodeEscapedUnicode-16      4.00 ± 0%      0.00       -100.00%  (p=0.000 n=10+10)
```

## 看源码

`binary.Read` 针对常用类型已经没有使用反射，但还是会分配多一个 slice

[https://github.com/golang/go/blob/83bc1ed3165e31d1fbeb1e6594373dc11b6ae0a4/src/encoding/binary/binary.go#L165](https://github.com/golang/go/blob/83bc1ed3165e31d1fbeb1e6594373dc11b6ae0a4/src/encoding/binary/binary.go#L165)

