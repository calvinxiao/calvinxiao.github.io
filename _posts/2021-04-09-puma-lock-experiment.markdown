---
layout: post
title:  "Puma Lock Experiment and write_nonblock"
date:   2021-04-09 22:03 +0800
categories: performance
---

I failed to optimize the lock contention, instead I found that `write_nonblock` is faster than `syswrite`.

`write_nonblock` will be used in [https://github.com/puma/puma/pull/2595](https://github.com/puma/puma/pull/2595)

Benchmark on master

```

# hello.ru

➜  puma git:(write_nonblock) wrk -c 4 -d 20 --latency http://localhost:9292
Running 20s test @ http://localhost:9292
  2 threads and 4 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.11ms    1.21ms  10.06ms   84.44%
    Req/Sec     2.54k   243.69     3.22k    68.50%
  Latency Distribution
     50%  674.00us
     75%    1.75ms
     90%    2.84ms
     99%    5.04ms
  100997 requests in 20.01s, 7.32MB read
Requests/sec:   5048.41
Transfer/sec:    374.69KB
➜  puma git:(write_nonblock) wrk -c 4 -d 20 --latency http://localhost:9292
Running 20s test @ http://localhost:9292
  2 threads and 4 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.15ms    1.31ms  10.04ms   84.35%
    Req/Sec     2.64k   251.73     3.33k    66.50%
  Latency Distribution
     50%  593.00us
     75%    1.81ms
     90%    3.03ms
     99%    5.45ms
  104966 requests in 20.01s, 7.61MB read
Requests/sec:   5245.66
Transfer/sec:    389.33KB

# realistic_response.ru

➜  puma git:(write_nonblock) ✗ wrk -c 4 -d 20 --latency http://localhost:9292
Running 20s test @ http://localhost:9292
  2 threads and 4 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.32ms  647.25us  10.93ms   68.37%
    Req/Sec     1.53k   125.52     3.80k    98.25%
  Latency Distribution
     50%    1.27ms
     75%    1.75ms
     90%    2.18ms
     99%    3.11ms
  61044 requests in 20.10s, 11.47GB read
Requests/sec:   3037.08
Transfer/sec:    584.30MB
➜  puma git:(master) wrk -c 4 -d 20 --latency http://localhost:9292
Running 20s test @ http://localhost:9292
  2 threads and 4 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.34ms  654.54us   9.49ms   68.31%
    Req/Sec     1.52k    64.47     1.69k    70.65%
  Latency Distribution
     50%    1.28ms
     75%    1.76ms
     90%    2.21ms
     99%    3.11ms
  60693 requests in 20.10s, 11.40GB read
Requests/sec:   3019.65
Transfer/sec:    580.94MB

```

After replacing `io.syswrite` to `io.write_nonblock`

```
# hello.ru
➜  puma git:(write_nonblock) wrk -c 4 -d 20 --latency http://localhost:9292
Running 20s test @ http://localhost:9292
  2 threads and 4 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.22ms    1.50ms   8.61ms   82.17%
    Req/Sec     3.46k   233.44     4.02k    66.00%
  Latency Distribution
     50%  148.00us
     75%    2.07ms
     90%    3.58ms
     99%    5.76ms
  137886 requests in 20.01s, 9.99MB read
Requests/sec:   6891.50
Transfer/sec:    511.48KB
➜  puma git:(write_nonblock) ✗ wrk -c 4 -d 20 --latency http://localhost:9292
Running 20s test @ http://localhost:9292
  2 threads and 4 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.23ms    1.52ms  10.96ms   82.14%
    Req/Sec     3.46k   243.02     4.21k    67.25%
  Latency Distribution
     50%  148.00us
     75%    2.08ms
     90%    3.61ms
     99%    5.83ms
  137651 requests in 20.00s, 9.98MB read
Requests/sec:   6880.93
Transfer/sec:    510.69KB

# realistic_response.ru

➜  puma git:(master) wrk -c 4 -d 20 --latency http://localhost:9292
Running 20s test @ http://localhost:9292
  2 threads and 4 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.21ms    2.75ms  22.98ms   82.41%
    Req/Sec     1.94k   167.95     2.37k    72.50%
  Latency Distribution
     50%  297.00us
     75%    3.71ms
     90%    6.50ms
     99%   10.77ms
  77109 requests in 20.01s, 14.49GB read
Requests/sec:   3853.68
Transfer/sec:    741.40MB
➜  puma git:(write_nonblock) wrk -c 4 -d 20 --latency http://localhost:9292
Running 20s test @ http://localhost:9292
  2 threads and 4 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.15ms    2.66ms  19.17ms   82.56%
    Req/Sec     1.91k   173.93     2.43k    67.75%
  Latency Distribution
     50%  303.00us
     75%    3.60ms
     90%    6.28ms
     99%   10.41ms
  76170 requests in 20.01s, 14.31GB read
Requests/sec:   3806.11
Transfer/sec:    732.26MB
```

Performance improvement:

- hello.ru, RPS: [5048.41, 5245.66] to [6891.50, 6880.93], **1.338x**
- realistic_response.ru, RPS: [3037.08, 3019.65] to [3853.68, 3806.11], **1.265x**