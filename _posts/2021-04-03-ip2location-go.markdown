---
layout: post
title:  "ip2location-go 性能优化"
date:   2021-04-03 00:19:00 +0800
categories: performance
---

## 背景

GoCN 推荐了 ip2location 项目 [【GoCN酷Go推荐】ip2location 解析 IP 地址库](https://mp.weixin.qq.com/s/yZiW2w8zA7eUqoefs_ewyA)，于是便去看看别人怎么写项目的，就算没找到性能优化点，多看点 github 好项目对自己写好代码也是有帮助。

然而，这个项目明显不是 gopher 写的，看这里 [#6](https://github.com/ip2location/ip2location-go/issues/6) ，还有这里[#3](https://github.com/ip2location/ip2location-go/issues/3)

我兴趣一下子就上来了，我也不是 gopher 啊。

## Approach 1

以前看过 ipip.net 的 Node.js 地址库，里面用的是线性查询，复杂度 `O(n)`，当时没有想过提（骗）PR，如果提 PR 的话就是用二分改写了，这个 ip2location 库用的就是二分，而说到性能优化，我一般采取的做法都是

1. 删代码，no code is faster than NO CODE
2. 算法复杂度优化
3. 缓存
4. 并发
5. do less work

看了下代码之后发现并没有什么代码可以删，1 可以排除

因为已经用了二分，理论上已经没有优化的空间，2 可以排除

缓存，MIT 6.172 有教优化 cache line，或者改用 cache friendly data structure，这里改底层数据结构可能是一个大工作，感觉对于这种要求高兼容性的项目并不是最高的需求，而且就算在 threshold 区间内使用线性查找，性能优化微乎其微，因为线性判断增加的 branching 带来的常数已经抵消高效使用 cache line 带来的性能提升，这个后面有说，所以在最后 3 也放弃了

并发是调用方是否决定使用的，该库提供只读功能，理论上是并发安全的，而且高并发也不会带来性能损失，因为没有 write。

所以，只能在 5 - do less work 上去做优化了

## Measure

> If you can't measure it, you can't improve it.

学习并使用 go test -bench . 测试。

fork 并创建 `before-performance` 分支 [https://github.com/calvinxiao/ip2location-go/tree/before-performance](https://github.com/calvinxiao/ip2location-go/tree/before-performance)

增加测试文件 `ip2location_test.go`

```
# go test -cpuprofile cpu.prof -memprofile mem.prof -bench BenchmarkGetAll -benchmem
BenchmarkGetAll1-16        14084             84946 ns/op           10161 B/op        696 allocs/op

# pprof -svg cpu.prof
```

查看 [svg](/assets/profile001.svg) 文件可以发现 `runtime mallocgc` 占用了 **18.92** 的时间，占用时间最多的是 `Pread` 读文件的功能，由于已经是使用了二分查找，理论上已经是最少的读文件次数了。


## Golang GC

我理解的 GC 包括分配内存和垃圾回收，Go 是带 GC 的，但并不表示可以随便分配内存否则会增加 GC 压力从而降低性能，对于一个 System Programmer 来说，能用 stack 就用 stack 吧。

mallocgc 的调用者：

1. readstr
2. readuint32
3. NewReader
4. big.NewInt

可以看 mem.prof 文件生成的 [svg](/assets/profile002.svg) 更清楚看到具体分配的内存的调用路径

期间看了一篇（还没看完）关于 Golang 性能优化的文章 [Writing and Optimizing Go code](https://github.com/dgryski/go-perfbook/blob/master/performance.md)

> binary.Read and binary.Write use reflection and are slow; do it by hand. (https://github.com/conformal/yubikey/commit/613e3b04ae2eeb78e6a19636b8ff8e9106d2e7bc)

那么巧了，代码里面用了 `binary.Read`

改！！

```
# go test -cpuprofile cpu.prof -memprofile mem.prof -bench BenchmarkGetAllOld -benchmem
BenchmarkGetAllOld-16              15841             75894 ns/op            5569 B/op        450 allocs/op

```

简单用 `binary.LittleEndian.Uint32` 替换 `binary.Read` 之后性能提升 **12%**，allocs 从 696/op 降到了 450/op

再看看 mem.prof 生成的 [svg](/assets/profile003.svg) 发现 `readuint32` 下面的 `binary.Read` 已经没有了，

## 尝试 Cache Line 优化

写了个 query2 函数，在二分区间剩下 4, 8, 16 ... 4096 的情况下使用线性查找，性能并没有优化，放弃了，感兴趣的看 [branch:performance-cache](https://github.com/calvinxiao/ip2location-go/tree/performance-cache)

## 替换 math/big

math/big 库提供了任意精度的大整数操作，但是 ipv6 占 16 字节，并不需要任意精度，去找了个 `uint128` 的库 [https://github.com/lukechampine/uint128](https://github.com/lukechampine/uint128) 替换所有 math/big 的操作

 当然，需要手动实现 `Not` 操作，lukechampine 的类库里面没有，但是对于看过 [Hacker's Delight](https://www.amazon.com/Hackers-Delight-2nd-Henry-Warren/dp/0321842685) 一点皮毛的系统程序员来说难度不大

 改动比较大，具体看分支 [performance-uint128](https://github.com/calvinxiao/ip2location-go/tree/performance-uint128)

 再次跑 benchmark

```
BenchmarkGetAllOld-16              17890             67029 ns/op            1776 B/op        276 allocs/op
```

运行次数 14084 -> 15841 -> 17890, allocs/op 696 -> 450 -> 276，Not bad

## 尝试 mmap

最后尝试了使用 mmap 预先加载文件到内存然后在内存中做二分查找，然后不是 gopher 写不出来，那么可以全部 ip 预先读出来放在数组里面也不是不行，毕竟现在最慢的已经是读文件操作 `Pread` 了，虽然有文件系统缓存，但是读一个数字还是要系统调用所以慢，不知道 mmap 之后是否会有系统调用？

## 总结

GC 优化，性能提升 **27%**, allocs/op 降低 **60%**

最后的 mem.prof [svg](/assets/profile004.svg), cpu.prof [svg](/assets/profile005.svg)

## TODO

1. 看如何做多分支 benchmark，我看 github action 是可以做的
2. 提个 issue 问问，看看人家喜不喜欢 PR，毕竟改动这么多还是要写测试的，官方库没有写测试。
