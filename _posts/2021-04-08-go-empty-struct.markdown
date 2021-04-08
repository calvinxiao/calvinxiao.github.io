---
layout: post
title:  "Go Empty Struct"
date:   2021-04-08 11:18:00 +0800
categories: performance
---

上期在尝试优化 Tidb 代码中 struct maligned 问题的时候发现有 struct 的 alignment 可以优化，具体是

`pushDownJoin` [https://github.com/pingcap/tidb/blob/8184e69eb7f53a024b8e79fb7eb9e1c2a174c8e4/planner/cascades/transformation_rules.go#L837](https://github.com/pingcap/tidb/blob/8184e69eb7f53a024b8e79fb7eb9e1c2a174c8e4/planner/cascades/transformation_rules.go#L837)
`PushSelDownJoin` [https://github.com/pingcap/tidb/blob/8184e69eb7f53a024b8e79fb7eb9e1c2a174c8e4/planner/cascades/transformation_rules.go#L939](https://github.com/pingcap/tidb/blob/8184e69eb7f53a024b8e79fb7eb9e1c2a174c8e4/planner/cascades/transformation_rules.go#L939)

```go
type pushDownJoin struct {
}

type PushSelDownJoin struct {
	baseRule
	pushDownJoin
}

```

`pushDownJoin` 是空 struct，baseRule struct 里面有一个指针，在 64 位机器上 baseRule 的大小是 8 字节，但是
`PushSelDownJoin` 大小确是 16 字节。

如果把 pushDownJoin 和 baseRule 互换位置，`PushSelDownJoin` 大小就变成了 8 字节。

官方有个 issue，在 struct 末尾增加了 `_ struct{}` 来防止 unkeyed literals，但是最后把 empty struct 放到了最前面来减少内存分配

> to prevent unkeyed literals. Trailing zero-sized field will take space.

[https://go-review.googlesource.com/c/go/+/15660/](https://go-review.googlesource.com/c/go/+/15660/)

如果空 struct 在最后，而且不 pad 的话，pushDownJoin 的内存地址可能会下一个数据的地址一样，造成各种问题，我猜的。

简单的测试代码告诉我们内存能省就省

```go
package main

import "testing"

type Empty struct{}

type Int64ThenEmpty struct {
	a uint64
	_ Empty
}

type EmptyThenInt64 struct {
	_ Empty
	a uint64
}

func BenchmarkEmpty1(b *testing.B) {
	for i := 0; i < b.N; i++ {
		c := make([]Int64ThenEmpty, 100000)
		c[0].a = 42
	}
}

func BenchmarkEmpty2(b *testing.B) {
	for i := 0; i < b.N; i++ {
		c := make([]EmptyThenInt64, 100000)
		c[0].a = 42
	}
}

// BenchmarkEmpty1-16          5545            213116 ns/op         1605664 B/op          1 allocs/op
// BenchmarkEmpty2-16         10000            117012 ns/op          802830 B/op          1 allocs/op

```

