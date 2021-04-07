---
layout: post
title:  "Go struct alignment"
date:   2021-04-07 18:54:00 +0800
categories: performance
---

I was trying to optimize structs in tidb, I need to find some critical paths first.

Foudn this issue [pingcap/tidb:11629](https://github.com/pingcap/tidb/pull/11629)

Also the `golangci-lint` tool is great
