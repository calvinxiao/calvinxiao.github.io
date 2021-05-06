---
layout: post
title:  "Why Golang String Plus Is Fast"
date:   2021-05-06 21:24 +0800
categories: performance
---

When working on performance, it's not enough to just find out why things are slow, 
understanding why things are fast is very important.

Here is my learning of why golang string plus is the fastest.

## 1. Benchmark

```golang
package main

import (
	"bytes"
	"strings"
	"testing"
)

func BenchmarkStringPlus(b *testing.B) {
	s1 := strings.Repeat("a", 100000)
	s2 := strings.Repeat("b", 100000)
	s3 := strings.Repeat("c", 100000)

	for i := 0; i < b.N; i++ {
		t := s1 + s2 + s3
		t = t
	}
}



func BenchmarkStringJoin(b *testing.B) {
	s1 := strings.Repeat("a", 100000)
	s2 := strings.Repeat("b", 100000)
	s3 := strings.Repeat("c", 100000)

	for i := 0; i < b.N; i++ {
		s := []string{s1, s2, s3}
		t := strings.Join(s, "")
		t = t
	}
}

func BenchmarkStringBuffer(b *testing.B) {
	s1 := strings.Repeat("a", 100000)
	s2 := strings.Repeat("b", 100000)
	s3 := strings.Repeat("c", 100000)

	for i := 0; i < b.N; i++ {
		var bsb bytes.Buffer
		bsb.WriteString(s1)
		bsb.WriteString(s2)
		bsb.WriteString(s3)
		t := bsb.String()
		t = t
	}
}

func BenchmarkStringBuilder(b *testing.B) {
	s1 := strings.Repeat("a", 100000)
	s2 := strings.Repeat("b", 100000)
	s3 := strings.Repeat("c", 100000)

	for i := 0; i < b.N; i++ {
		var sb strings.Builder
		sb.WriteString(s1)
		sb.WriteString(s2)
		sb.WriteString(s3)
		t := sb.String()
		t = t
	}
}

## 2. Result

```
BenchmarkStringPlus-16             19687             59930 ns/op          303122 B/op          1 allocs/op
BenchmarkStringJoin-16             18229             66459 ns/op          303123 B/op          1 allocs/op
BenchmarkStringBuffer-16            6722            155969 ns/op          712756 B/op          3 allocs/op
BenchmarkStringBuilder-16           8328            128893 ns/op          655401 B/op          3 allocs/op
```

## 3. `disasm` Memory Profile

# go tool pprof mem.prof
# disasm BenchmarkStringPlus

         .          .     50c163: MOVQ CX, 0x30(SP)
    8.69GB     8.69GB     50c168: CALL runtime.concatstring3(SB)          ;test_cpu.BenchmarkStringPlus string_concat_test.go:15
         .          .     50c16d: MOVQ 0x60(SP), AX                       ;string_concat_test.go:14

```

Notice that string plus is using `runtime.concatstring3`, a special case to concat 3 strings, checking out
 the [golang source code](https://github.com/golang/go/blob/2ebe77a2fda1ee9ff6fd9a3e08933ad1ebaea039/src/runtime/string.go) currently there are `concatstring{2, 3, 4, 5}`, let's see how performance turns when 
concating 6 strings.

Using similar code to do 6 strings concat, and the size is changed from `100000` to `50000`, you know why.


```
BenchmarkStringPlus-16             20656             59404 ns/op          303121 B/op          1 allocs/op
BenchmarkStringPlus6-16            20262             60456 ns/op          303123 B/op          1 allocs/op
```

So, performance is slightly down, but not much.

The reason why string concatenation is fast is go doesn't allocate memory when doint every `+` action, instead 
putting all the string to a string slice and calculate the final length, no matter how many `+` in one line, 
go only allocates once.