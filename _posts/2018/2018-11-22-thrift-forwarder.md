---
layout: post
title: RPC forwarding for Thrift
tags:
- thrift
- csharp
---

I started this post some time ago.  I was convinced we could improve on the current implementation and that would make for an interesting read.  Instead, we'll likely drop Apache Thrift so the improved version will never happen.  This might still be valuable (or interesting, at least) to someone, so here's the current version.

We have a need to forward Thrift RPC calls in a message-agnostic manner.  This is useful for various reasons:

- Bridge different transports/protocols (e.g. HTTP<->named pipe)
- Message routing/filtering/logging or other processing

## Message Processor

A Thrift message processor is created by deriving from [`ITAsyncProcessor`](https://github.com/apache/thrift/blob/master/lib/netcore/Thrift/ITAsyncProcessor.cs):
```csharp
public class ForwardingProcessor : ITAsyncProcessor
{
    public ForwardingProcessor(TProtocol prot)
        : this(prot, prot)
    { }

    public ForwardingProcessor(TProtocol fromDest, TProtocol toDest)
    {
        FromDest = fromDest;
        ToDest = toDest;
    }

    public Task<bool> ProcessAsync(TProtocol fromSrc, TProtocol toSrc)
    {
        return ProcessAsync(fromSrc, toSrc, CancellationToken.None);
    }

    TProtocol FromDest; // Read from destination
    TProtocol ToDest; // Write to destination
}
```

Usage:
```csharp
void Register(TMultiplexedProcessor multiplexor)
{
    var protocol = new ZeroMqProtocol(brokerAddress);
    protocol.Channel = serviceName;
    var processor = new ForwardingProcessor(protocol);
    multiplexor.RegisterProcessor("SER_" + serviceName, processor);
}
```

The [TMultiplexedProcessor](https://github.com/apache/thrift/blob/master/lib/netcore/Thrift/TMultiplexedProcessor.cs) could be, for example, the same processor we're using with [ASP.NET Core]({% post_url /2018/2018-08-24-aspnetcore-thrift %}).  `ZeroMqProtocol` is our [ZeroMQ transport for Thrift]({% post_url /2018/2018-09-07-zeromq-thrift %}).  This would have the effect of bridging HTTP and ZeroMQ.

The resulting message flow is:

1. HTTP client connects and [sends message like `SER_L10NSERVICE:GetCurrentLanguage`]({% post_url /2018/2018-08-30-rust-thrift %})
1. ASP.[]()NET Core dispatches it to `THttpHandler` middleware (`TMultiplexedProcessor` instance using `TJSONProtocol`)
1. It's mapped to `ForwardingProcessor` instance whose `ProcessAsync()` is called
1. RPC request is written to `ZeroMqProtocol`

## Long Way

Our first (and current) implementation deserializes the message from one `TProtocol` and serializes it to another:

```csharp
public async Task<bool> ProcessAsync(TProtocol fromSrc, TProtocol toSrc, CancellationToken token)
{
    try
    {
        TMessage msg = await fromSrc.ReadMessageBeginAsync();
        // Save sequence id to make sure our response contains the same as the request
        var seqId = msg.SeqID;

        if (!IsForwarded())
        {
            return false;
        }

        await ToDest.WriteMessageBeginAsync(msg);

        // Forward RPC args
        await forwardStruct(fromSrc, ToDest, token);
        await fromSrc.ReadMessageEndAsync();
        await ToDest.WriteMessageEndAsync();
        await ToDest.Transport.FlushAsync();

        // Return RPC result
        var resp = await FromDest.ReadMessageBeginAsync();
        resp.SeqID = seqId;
        await toSrc.WriteMessageBeginAsync(resp);
        if (msg.Type == TMessageType.Exception)
        {
            var x = await TApplicationException.ReadAsync(FromDest, token);
            await x.WriteAsync(toSrc, token);
        }
        else
        {
            await forwardStruct(FromDest, toSrc, token);
        }

        await FromDest.ReadMessageEndAsync();
        await toSrc.WriteMessageEndAsync();
        await toSrc.Transport.FlushAsync();
    }
    catch (IOException)
    {
        return false;
    }
    return true;
}
```

The `IsForwarded()` function is a placeholder for filtering logic we may want to do.  This is currently hard-coded to return `true`, but if/when we get around to make use of it the early return may need to use `TProtocolUtil.SkipAsync()` and/or write a response of some kind.


### Forward TStruct

```csharp
/// <summary>
/// Reads <see cref="TStruct"/> from <paramref name="iprot"/> and writes to <paramref name="oprot"/>.
/// </summary>
/// <param name="iprot"></param>
/// <param name="oprot"></param>
async Task forwardStruct(TProtocol iprot, TProtocol oprot, CancellationToken token)
{
    iprot.IncrementRecursionDepth();
    oprot.IncrementRecursionDepth();

    try
    {
        TField field;
        var tstruct = await iprot.ReadStructBeginAsync();
        await oprot.WriteStructBeginAsync(tstruct);

        while (true)
        {
            field = await iprot.ReadFieldBeginAsync();
            
            if (field.Type == TType.Stop)
            {
                // NB: With Stop field don't WriteFieldBegin() or End()
                await oprot.WriteFieldStopAsync();
                break;
            }

            await oprot.WriteFieldBeginAsync(field);
            await forwardElement(field.Type, iprot, oprot, token);
            await Task.WhenAll(
                iprot.ReadFieldEndAsync(),
                oprot.WriteFieldEndAsync()
                );
        }
        await Task.WhenAll(
            iprot.ReadStructEndAsync(),
            oprot.WriteStructEndAsync()
            );
    }
    finally
    {
        iprot.DecrementRecursionDepth();
        oprot.DecrementRecursionDepth();
    }
}
```

Again, this code is heavily based off the generated source code.

The handling of `TType.Stop` which eschews `ReadFieldEnd()` as well as `WriteFieldBegin|End()`.

`await Task.WhenAll(task, ...)` uses [`WhenAll()`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.whenall) to overlap independent operations.

### Forward TType

```csharp
async Task forwardElement(TType elementType, TProtocol iprot, TProtocol oprot, CancellationToken token)
{
    switch (elementType)
    {
        case TType.Bool:
            var vBool = await iprot.ReadBoolAsync();
            await oprot.WriteBoolAsync(vBool);
            break;
        case TType.Byte:
            var vByte = await iprot.ReadByteAsync();
            await oprot.WriteByteAsync(vByte);
            break;
        case TType.Double:
            var vDouble = await iprot.ReadDoubleAsync();
            await oprot.WriteDoubleAsync(vDouble);
            break;
        case TType.I16:
            var vI16 = await iprot.ReadI16Async();
            await oprot.WriteI16Async(vI16);
            break;
        case TType.I32:
            var vI32 = await iprot.ReadI32Async();
            await oprot.WriteI32Async(vI32);
            break;
        case TType.I64:
            var vI64 = await iprot.ReadI64Async();
            await oprot.WriteI64Async(vI64);
            break;
        case TType.List:
            var list = await iprot.ReadListBeginAsync();
            await oprot.WriteListBeginAsync(list);
            for (int i = 0; i < list.Count; ++i)
            {
                await forwardElement(list.ElementType, iprot, oprot, token);
            }
            await Task.WhenAll(
                iprot.ReadListEndAsync(),
                oprot.WriteListEndAsync()
                );
            break;
        case TType.Map:
            var map = await iprot.ReadMapBeginAsync();
            await oprot.WriteMapBeginAsync(map);
            for (int i = 0; i < map.Count; ++i)
            {
                await forwardElement(map.KeyType, iprot, oprot, token);
                await forwardElement(map.ValueType, iprot, oprot, token);
            }
            await Task.WhenAll(
                iprot.ReadMapEndAsync(),
                oprot.WriteMapEndAsync()
                );
            break;
        case TType.Set:
            var set = await iprot.ReadSetBeginAsync();
            await oprot.WriteSetBeginAsync(set);
            for (int i = 0; i < set.Count; ++i)
            {
                await forwardElement(set.ElementType, iprot, oprot, token);
            }
            await Task.WhenAll(
                iprot.ReadSetEndAsync(),
                oprot.WriteSetEndAsync()
                );
            break;
        case TType.String:
            var vString = await iprot.ReadStringAsync(token);
            await oprot.WriteStringAsync(vString, token);
            break;
        case TType.Struct:
            await forwardStruct(iprot, oprot, token);
            break;
        case TType.Void:
            throw new System.NotImplementedException();
        default:
            // Unexpected type.  More robust to skip than throw exception
            await TProtocolUtil.SkipAsync(iprot, elementType, token);
            break;
    }
}
```

A big ol' `switch` statement 1:1 mapping of `TType` to `TProtocol.(Read|Write)*Async()` methods.

`TType.List`, `TType.Map`, and `TType.Set` are all similar.  Begin/End Read/Write pairs between which we recurse into `forwardElement()` for each element of the collection.

`TType.Struct` recurses into [`forwardStruct()`](#forward-tstruct) to handle nested structures.

`TType.Void` throws an exception because I suspect it cannot happen:
- We have no examples of a `void` struct member
- [`TProtocolUtil.SkipAsync()`](https://github.com/apache/thrift/blob/af7ecd6a2b15efe5c6b742cf4a9ccb31bcc1f362/lib/netcore/Thrift/Protocols/Utilities/TProtocolUtil.cs#L27) implementation doesn't handle `TType.Void`

`default` case currently cannot happen as we handle all `TType` values.  But we `TProtocolUtil.SkipAsync()` in case someone adds a type.

## Short Way?

We've mentioned a component that already does what we want- `TMultiplexedProcessor`; it receives a message, processes the service header and forwards it to a registered processor.

Inside [`TMultiplexedProcessor.ProcessAsync()`](https://github.com/apache/thrift/blob/af7ecd6a2b15efe5c6b742cf4a9ccb31bcc1f362/lib/netcore/Thrift/TMultiplexedProcessor.cs#L41):
```csharp
// Create a new TMessage, removing the service name
var newMessage = new TMessage(
    message.Name.Substring(serviceName.Length + TMultiplexedProtocol.Separator.Length),
    message.Type,
    message.SeqID);

// Dispatch processing to the stored processor
return
    await
        actualProcessor.ProcessAsync(new StoredMessageProtocol(iprot, newMessage), oprot,
            cancellationToken);
```

Where [`StoredMessageProtocol` is](https://github.com/apache/thrift/blob/af7ecd6a2b15efe5c6b742cf4a9ccb31bcc1f362/lib/netcore/Thrift/TMultiplexedProcessor.cs#L122):
```csharp
private class StoredMessageProtocol : TProtocolDecorator
{
    readonly TMessage _msgBegin;

    public StoredMessageProtocol(TProtocol protocol, TMessage messageBegin)
        : base(protocol)
    {
        _msgBegin = messageBegin;
    }

    public override async Task<TMessage> ReadMessageBeginAsync(CancellationToken cancellationToken)
    {
        if (cancellationToken.IsCancellationRequested)
        {
            return await Task.FromCanceled<TMessage>(cancellationToken);
        }

        return _msgBegin;
    }
}
```

We suspect there's a way to leverage this to implement `ForwardingProcessor.ProcessAsync()` without deserializing then reserializing each message.  Problem is that as a `TProtocolDecorator` it's defining a static pipeline of sorts, but it's not actively pulling the messages through such that they can be written out somewhere else.

Sadly, this is as far as we will take things.
