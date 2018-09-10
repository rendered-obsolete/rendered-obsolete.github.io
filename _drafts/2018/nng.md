---
layout: post
title: nng
tags:
- nng
- zeromq
- nanomsg
- csharp
---

In [another post]() I mention how we had used [ZeroMQ](http://zeromq.org/) (actually, [NetMQ](https://github.com/zeromq/netmq/)).  [NetMQ.WebSockets](https://github.com/NetMQ/NetMQ.WebSockets).

## nanomsg

Input stuff (C++) moved to [nanomsg](https://github.com/nanomsg/nanomsg) for IPC/named-pipe support but also supports [WebSocket](https://nanomsg.org/v1.1.4/nn_ws.html).

Did some initial investigation [forking](https://github.com/zplus/NNanomsg) [NNanomsg](https://github.com/mhowlett/NNanomsg) to convert it to .Net Standard and fix some small issues.

## nng

[NNG](https://github.com/nanomsg/nng).

[forking](https://github.com/zplus/csnng) [csnng](https://github.com/zplus/csnng).  Again, to convert it to .Net Standard and fix some issues.
