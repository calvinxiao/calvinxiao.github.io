---
layout: post
title:  "Aware Of Cacheline"
date:   2021-04-26 23:56 +0800
categories: performance
---

Most Golang tutorials will teach you how to use mutex properly.

Here is an example explaining how to effectively utilize modern hardware.

## Cacheline

Most cpu has cacheline the size of 64 bytes, when cpu fetch data from memory
to L1/L2 caches, it fetches the size of a cacheline, so keeping your data nice
and compact would increase read performance. 

Not the case if you want to update them concurrently.

## False Sharing

You can search `cacheline + false sharing` online to learn what they are, articles
are very good at explaining them.

Below is the simple Golang code showing the false sharing impact.

To use the full code snippet, check out [here](https://gist.github.com/calvinxiao/e5c572cb4aeeb62102941bcaaefe9d3d).

```go
package main

import (
	"sync"
	"sync/atomic"
	"testing"
)

const THREADS int = 8
const LOOP int = 100000

func BenchmarkCachelineMutex(b *testing.B) {
	for run := 0; run < b.N; run++ {
		total := 0
		var wg sync.WaitGroup
		wg.Add(THREADS)
		var mu sync.Mutex
		for i := 0; i < THREADS; i++ {
			go func() {
				for j := 0; j < LOOP; j++ {
					mu.Lock()
					total++
					mu.Unlock()
				}
				wg.Done()
			}()
		}
		wg.Wait()
	}
}

func BenchmarkCachelineAtomic(b *testing.B) {
	for run := 0; run < b.N; run++ {

		var total = new(int32)
		var wg sync.WaitGroup
		wg.Add(THREADS)
		for i := 0; i < THREADS; i++ {
			go func() {
				for j := 0; j < LOOP; j++ {
					atomic.AddInt32(total, 1)
				}
				wg.Done()
			}()
		}
		wg.Wait()
	}
}

func BenchmarkCachelineArray1(b *testing.B) {
	for run := 0; run < b.N; run++ {

		var total [2048]int
		var wg sync.WaitGroup
		wg.Add(THREADS)
		for i := 0; i < THREADS; i++ {
			go func(idx int) {
				for j := 0; j < LOOP; j++ {
					total[idx]++
				}
				wg.Done()
			}(i * 1)
		}
		wg.Wait()
	}
}

func BenchmarkCachelineArray2(b *testing.B) {
	for run := 0; run < b.N; run++ {

		var total [2048]int
		var wg sync.WaitGroup
		wg.Add(THREADS)
		for i := 0; i < THREADS; i++ {
			go func(idx int) {
				for j := 0; j < LOOP; j++ {
					total[idx]++
				}
				wg.Done()
			}(i * 2)
		}
		wg.Wait()
	}
}

// strip BenchmarkCachelineArray2 to BenchmarkCachelineArray13

func BenchmarkCachelineArray14(b *testing.B) {
	for run := 0; run < b.N; run++ {

		var total [2048]int
		var wg sync.WaitGroup
		wg.Add(THREADS)
		for i := 0; i < THREADS; i++ {
			go func(idx int) {
				for j := 0; j < LOOP; j++ {
					total[idx]++
				}
				wg.Done()
			}(i * 14)
		}
		wg.Wait()
	}
}


```

Run command `go test cacheline_test.go -run=BenchmarkCacheline -bench=BenchmarkCacheline -benchmem`
and you will get the benchmark statistic:

```
BenchmarkCachelineMutex-16                    30          40018997 ns/op            1256 B/op          6 allocs/op
BenchmarkCachelineAtomic-16                  145           8215842 ns/op             226 B/op          2 allocs/op
BenchmarkCachelineArray1-16                  442           2677674 ns/op           16456 B/op          2 allocs/op
BenchmarkCachelineArray2-16                  849           1436529 ns/op           16406 B/op          2 allocs/op
BenchmarkCachelineArray3-16                 1143           1034337 ns/op           16401 B/op          2 allocs/op
BenchmarkCachelineArray4-16                 1976            621679 ns/op           16408 B/op          2 allocs/op
BenchmarkCachelineArray5-16                 1790            699448 ns/op           16400 B/op          2 allocs/op
BenchmarkCachelineArray6-16                 2383            458006 ns/op           16401 B/op          2 allocs/op
BenchmarkCachelineArray7-16                 2215            563357 ns/op           16400 B/op          2 allocs/op
BenchmarkCachelineArray8-16                 5644            211981 ns/op           16405 B/op          2 allocs/op
BenchmarkCachelineArray9-16                 4479            281926 ns/op           16400 B/op          2 allocs/op
BenchmarkCachelineArray10-16                6565            211207 ns/op           16403 B/op          2 allocs/op
BenchmarkCachelineArray11-16                5283            220976 ns/op           16400 B/op          2 allocs/op
BenchmarkCachelineArray12-16                6944            189071 ns/op           16400 B/op          2 allocs/op
BenchmarkCachelineArray13-16                5514            219693 ns/op           16400 B/op          2 allocs/op
BenchmarkCachelineArray14-16                5882            189697 ns/op           16400 B/op          2 allocs/op
BenchmarkCachelineArray15-16                5656            236692 ns/op           16400 B/op          2 allocs/op
BenchmarkCachelineArray16-16                5886            214639 ns/op           16400 B/op          2 allocs/op
BenchmarkCachelineArray17-16                5835            207106 ns/op           16400 B/op          2 allocs/op
PASS
ok      command-line-arguments  25.139s
```

Most of the time, using mutex is good enough to avoid race condition. Yet using atomic(lock free) 
data structures are usually faster, in this case **4.8x**

In methods `BenchmarkCachelineArray%d` I use an array to store different result for different goroutine.

Take `BenchmarkCachelineArray8` for example, the 1st goroutine update total[0], the 2nd one update total[8], `int` is 4 bytes, so total[0] and total[8] is 32(31?) bytes away.


`BenchmarkCachelineArray1` is the slowest one due to false sharing, because as long as the 1st goroutine write data to total[0], it will invalidate other goroutines' cache, because total[0] to total[7] is in one cacheline.

Benchmark result can vary due to system scheduling, on my machine, the next run shows `BenchmarkCachelineArray14` is the fastest one.

Also you should adjust `THREADS` according to the cpu cores on your machine.

## Reference

- [MIT 6.172 - 6. Multicore Programming](https://youtu.be/dx98pqJvZVk?list=PLUl4u3cNGP63VIBQVWguXxZZi0566y7Wf&t=606)
- [Understanding CPU Microarchitecture to Increase Performance - Alex Blewitt](https://youtu.be/rglmJ6Xyj1c?t=749)
- [Episode 5.9 - Elimination of False Cache Line Sharing](https://www.youtube.com/watch?v=h58X-PaEGng)

