---
layout: post
title: Apache Thrift over ZeroMQ
tags:
- thrift
- zeromq
- 0mq
- netmq
---

The background service portion of our client ([service](), [layer0](), [layer1]()) originally used a combination of Thrift transports for RPC and [ZeroMQ](http://zeromq.org/) (via [NetMQ](https://github.com/zeromq/netmq/)).  Consequently, we had a ton of ports/connections.

[Multiplexing the thrift clients]() helped.  Our technical director had the idea of using ZeroMQ itself as the transport for thrift.  In effect, multiplexing all our communicating agents over ZeroMQ.

[Majordomo protocol](https://rfc.zeromq.org/spec:7/MDP/) using the [NetMQ sample](https://github.com/NetMQ/Samples/tree/master/src/Majordomo)

## Transport

```csharp
public class ZeroMqTransport : TStreamTransport
{
    public string Channel { get; set; }

    private MDPClient Client { get; set; }

    internal ZeroMqTransport(MDPClient client)
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

`Common.MemoryPool.GetStream()` is a wrapper around [`Microsoft.IO.RecyclableMemoryStreamManager.GetStream()`](http://www.philosophicalgeek.com/2015/02/06/announcing-microsoft-io-recycablememorystream/).

## Protocol

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

This greatly simplified our architecture because there's only one source of connections: ZeroMQ.