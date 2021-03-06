---
layout: post
title: Apache Thrift over ZeroMQ
tags:
- thrift
- zeromq
- 0mq
- netmq
---

The background service portion of our client ([service]({% post_url /2018/2018-08-21-windows-services %}), [layer0]({% post_url /2018/2018-08-28-layer0 %}), [layer1]({% post_url /2018/2018-09-04-layer1 %})) originally used a combination of Thrift transports for RPC and [ZeroMQ](http://zeromq.org/) (via [NetMQ](https://github.com/zeromq/netmq/)) for [pub/sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern).  Consequently, we had a ton of ports/connections.

[Multiplexing the thrift clients]({% post_url /2018/2018-08-30-rust-thrift %}) with [TMultiplexedProtocol](https://github.com/apache/thrift/blob/master/lib/netcore/Thrift/Protocols/TMultiplexedProtocol.cs) helped.  Our technical director had the idea of using ZeroMQ itself as the transport for thrift.  In effect, multiplexing all our communicating agents over ZeroMQ.

We also leverage the [Majordomo protocol](https://rfc.zeromq.org/spec:7/MDP/) (from the [NetMQ samples](https://github.com/NetMQ/Samples/tree/master/src/Majordomo)) to provide additional service-oriented functionality atop ZeroMQ.

## Transport

```csharp
public class ZeroMqTransport : TStreamClientTransport
{
    public string Channel { get; set; }
    private MDPClient Client { get; set; }

    public ZeroMqTransport(MDPClient client)
    {
        Client = client;

        InputStream = null; // only set input stream after received data
        OutputStream = Common.MemoryPool.GetStream(); // set output stream so we can write message into it
    }

    public override async Task FlushAsync(CancellationToken token)
    {
        await base.FlushAsync(token);

        try
        {
            var ostream = OutputStream as MemoryStream;
            if (ostream.Length <= 0)
                return;

            InputStream = null;
            NetMQMessage req = new NetMQMessage();
            req.Append(ostream.ToArray());

            // TODO: rep could be null if time out happens
            var rep = Client.Send(Channel, req);

            InputStream = rep == null ? null : Common.MemoryPool.GetStream(rep.Last.Buffer);
            OutputStream = Common.MemoryPool.GetStream();
        }
        catch (Exception ex)
        {
            Logging.Logger.Log($"zmq transport flush: {ex.Message}", Logging.LogLevel.Warn);
            Logging.Logger.Log($"zmq transport flush: {ex.Message} \n {ex.StackTrace}", Logging.LogLevel.Debug);
        }
    }
    
}
```

We're overriding `FlushAsync()`.  If you take a look at code generated by thrift:
```csharp
await oprot.WriteMessageBeginAsync(..., cancellationToken);
// Write the message...
await oprot.WriteMessageEndAsync(cancellationToken);
await oprot.Transport.FlushAsync(cancellationToken);
```

The transport's `FlushAsync()` method is called at the end of every message.  We use that to handle a complete message:

1. Construct `NetMQMessage` containing serialized thrift message
1. `MDPClient.Send()` over ZeroMQ
1. If there is a reply, set `TStreamClientTransport.InputStream`

Lastly, `Common.MemoryPool.GetStream()` is a wrapper around [`Microsoft.IO.RecyclableMemoryStreamManager.GetStream()`](http://www.philosophicalgeek.com/2015/02/06/announcing-microsoft-io-recycablememorystream/).

## Protocol

The protocol isn't particularly interesting, it inherits behaviour from [`TBinaryProtocol`](https://github.com/apache/thrift/blob/master/lib/netcore/Thrift/Protocols/TBinaryProtocol.cs) and wraps the `MDPClient` and `ZeroMqTransport`.  Here it is for completeness:
```csharp
public class ZeroMqProtocol : TBinaryProtocol
{
    private MDPClient syncClient = null;
    private ZeroMqTransport zmqTransport = null;

    public ZeroMqProtocol(string brokerAddr, byte[] ClientID) : base(null)
    {
        syncClient = new MDPClient(brokerAddr, ClientID);
        zmqTransport = new ZeroMqTransport(syncClient);
        Trans = zmqTransport;
    }

    protected override void Dispose(bool disposing)
    {
        if (zmqTransport != null)
        {
            zmqTransport.Close();
            zmqTransport.Dispose();
            zmqTransport = null;
        }
        if (syncClient != null)
        {
            syncClient.Dispose();
            syncClient = null;
        }
        base.Dispose(disposing);
    }

    public string Channel
    {
        set { zmqTransport.Channel = value; }
    }
}
```

## Footnotes

This may or may not be the correct way to do this.  This is one of our earlier pieces of code when we were still cutting our teeth on both ZeroMQ and Thrift.

In any case, it greatly simplified our architecture because there's only one source of connections: ZeroMQ.