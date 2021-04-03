---
layout: post
title:  "Reduce puma_parser struct padding"
date:   2021-04-05 03:23:00 +0800
categories: performance
---

[puma/puma#2590](https://github.com/puma/puma/pull/2590)

估计是人生第一次优化 C struct 的 alignment issue.