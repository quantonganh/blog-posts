---
title: "libp2p performance benchmarking"
date: 2023-10-27
description:
categories:
- Programming
tags:
- libp2p
- rust
- dnctl
- pfctl
---
I invested some time in studying [QUIC](https://docs.libp2p.io/concepts/transports/quic/) and [libp2p](https://github.com/libp2p/rust-libp2p)
In this article, I will show you how to benchmark the network transfer using [perf](https://github.com/libp2p/rust-libp2p/tree/master/protocols/perf) module for 2 scanerios:

- high latency, no packet loss
- low latency but high packet loss

Let's print help first:

```
     Running `/Users/quantong/Code/github.com/libp2p/rust-libp2p/target/debug/perf -h`
Usage: perf [OPTIONS]

Options:
      --server-address <SERVER_ADDRESS>
      --transport <TRANSPORT>
      --upload-bytes <UPLOAD_BYTES>
      --download-bytes <DOWNLOAD_BYTES>
      --run-server                       Run in server mode
  -h, --help                             Print help  
```

Run in server mode:

```
     Running `/Users/quantong/Code/github.com/libp2p/rust-libp2p/target/debug/perf --server-address '127.0.0.1:8080' --run-server`
[2023-10-27T07:17:09.543Z INFO  libp2p_swarm] Local peer id: 12D3KooWPxXR3Me5pS7UjRg2gPVk9A7Se4nhEVBT9GPifjAwpiWX
[2023-10-27T07:17:09.545Z INFO  perf] Listening on /ip4/127.0.0.1/tcp/8080
[2023-10-27T07:17:09.545Z INFO  perf] Listening on /ip4/127.0.0.1/udp/8080/quic-v1
```

In another terminal, try to upload/download 100MB:

```
     Running `/Users/quantong/Code/github.com/libp2p/rust-libp2p/target/debug/perf --server-address '127.0.0.1:8080' --transport tcp --upload-bytes 104857600 --download-bytes 104857600`
[2023-10-27T07:21:14.922Z INFO  libp2p_swarm] Local peer id: 12D3KooWCwLib8hXRpVYNZFFnjBpQeqdKfFNgQUTj4mpJ2qyGdiZ
[2023-10-27T07:21:14.922Z INFO  perf] start benchmark: custom
[2023-10-27T07:21:14.930Z INFO  perf] established connection in 0.0076 s
[2023-10-27T07:21:15.935Z INFO  perf] 1.005029375 s uploaded 45.86 MiB downloaded 0 B (365.02 Mbit/s)
{"type":"intermediate","timeSeconds":1.005029375,"uploadBytes":48083968,"downloadBytes":0}
[2023-10-27T07:21:16.940Z INFO  perf] 1.005100667 s uploaded 46.42 MiB downloaded 0 B (369.45 Mbit/s)
{"type":"intermediate","timeSeconds":1.005100667,"uploadBytes":48671744,"downloadBytes":0}
[2023-10-27T07:21:17.945Z INFO  perf] 1.005020208 s uploaded 7.73 MiB downloaded 38.97 MiB (371.73 Mbit/s)
{"type":"intermediate","timeSeconds":1.005020208,"uploadBytes":8101888,"downloadBytes":40865792}
[2023-10-27T07:21:18.950Z INFO  perf] 1.005063875 s uploaded 0 B downloaded 46.87 MiB (373.07 Mbit/s)
{"type":"intermediate","timeSeconds":1.005063875,"uploadBytes":0,"downloadBytes":49146880}
[2023-10-27T07:21:19.253Z INFO  perf] uploaded 100.00 MiB in 2.1779 s (367.33 Mbit/s), downloaded 100.00 MiB in 2.1454 s (372.89 Mbit/s)
{"type":"final","timeSeconds":4.331209541,"uploadBytes":104857600,"downloadBytes":104857600}  
```

To simulate network latency and packet loss, we can use [pfctl](https://man.freebsd.org/cgi/man.cgi?query=pfctl) and [dnctl](https://man.freebsd.org/cgi/man.cgi?query=dnctl)

First, enable the packet filter:

```sh
sudo pfctl -E
```

Then create a custom anchor in pf:

```sh
$ cat /etc/pf.conf && echo "dummynet-anchor \"libp2p\"" && echo "anchor \"libp2p\"" | sudo pfctl -f -
```

Pipe the traffic to dummynet:

```sh
$ echo "dummynet in quick proto tcp from any to any port 8080 pipe 1" | sudo pfctl -a libp2p -f -
```

Simulate the network latency:

```sh
$ sudo dnctl pipe 1 config delay 10ms
```

Benchmark again:

```
Running `/Users/quantong/Code/github.com/libp2p/rust-libp2p/target/debug/perf --server-address '127.0.0.1:8080' --transport tcp --upload-bytes 104857600 --download-bytes 104857600`
[2023-10-27T07:58:43.515Z INFO  perf] uploaded 100.00 MiB in 8.4413 s (94.77 Mbit/s), downloaded 100.00 MiB in 8.2188 s (97.34 Mbit/s)
{"type":"final","timeSeconds":16.689302458,"uploadBytes":104857600,"downloadBytes":104857600}
```

Simulate the packet loss:

```sh
$ sudo dnctl pipe 1 config plr 0.01
```

```sh
$ sudo dnctl list
00001: unlimited    0 ms   50 sl.plr 0.010000 1 queues (1 buckets) droptail
    mask: 0x00 0x00000000/0x0000 -> 0x00000000/0x0000
BKT Prot ___Source IP/port____ ____Dest. IP/port____ Tot_pkt/bytes Pkt/Byte Drp
    mask: 0x00 0x00000000/0x0000 -> 0x00000000/0x0000
BKT Prot ___Source IP/port____ ____Dest. IP/port____ Tot_pkt/bytes Pkt/Byte Drp
  0 tcp        127.0.0.1/63691       127.0.0.1/8080  1011241 490303519  0    0 9463
```

```
Running `/Users/quantong/Code/github.com/libp2p/rust-libp2p/target/debug/perf --server-address '127.0.0.1:8080' --transport tcp --upload-bytes 104857600 --download-bytes 104857600`
[2023-10-27T08:33:58.098Z INFO  perf] uploaded 100.00 MiB in 9.1012 s (87.90 Mbit/s), downloaded 100.00 MiB in 2.2653 s (353.16 Mbit/s)
{"type":"final","timeSeconds":11.3747555,"uploadBytes":104857600,"downloadBytes":104857600}
```
