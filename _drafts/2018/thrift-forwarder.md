---
layout: post
title: RPC forwarding for Thrift
tags:
- thrift
---

- Bridge different transports/protocols (e.g. HTTP<->named pipe)
- Message routing/filtering

First, I'll go over the long way because it's interesting to understand how thrift works.  Then, I'll cover the short way.

## ITAsyncProcessor

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

Mostly boilerplate implementing a `ITAsyncProcessor`.

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

`ZeroMqProtocol` is [our ZeroMQ transport for Thrift](zerqmq-thrift).

TMultiplexedProcessor is [ASP.NET Core]({% post_url /2018/2018-08-24-aspnetcore-thrift %}).
1. HTTP client connects and [sends message like `SER_L10NSERVICE:GetCurrentLanguage`](rust-thrift)
1. ASP.[]()NET Core dispatches it to `THttpHandler` middleware (`TMultiplexedProcessor` instance using `TJSONProtocol`)
1. It's mapped to `ForwardingProcessor` instance whose `ProcessAsync()` is called
1. RPC request is written to `ZeroMqProtocol`

## ProcessAsync

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


## TStruct

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

`await Task.WhenAll(task, ...)` uses `WhenAll()` to overlap independent operations.

## TType

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

`TType.Struct` recurses into [`forwardStruct()`](#tstruct) to handle nested structures.

`TType.Void` throws an exception because I suspect it cannot happen:
- We have no examples of a `void` struct member
- [`TProtocolUtil.SkipAsync()`]() implementation doesn't handle `TType.Void`

`default` case currently cannot happen as we handle all `TType` values.  But we `TProtocolUtil.SkipAsync()` in case someone adds a type.

## Short Way

As it turns out, we've mentioned a component that already does what we want- `TMultiplexedProcessor`; it receives a message, processes the service header and forwards it to a registered processor.

Inside `TMultiplexedProcessor.ProcessAsync()`:
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

Where `StoredMessageProtocol` is:
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