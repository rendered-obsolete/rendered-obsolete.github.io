---
layout: post
title: Migrating to C# 8, or- How I Learned to Stop Worrying and Love .Net Core
tags:
- csharp
- dotnet
---

Gather around the fire and let me tell you a story...

A long time ago (ok, 4 years ago) in a strange, faraway land (ok, China), I made a number of "outlandish" decisions that resulted in much wailing and gnashing of teeth.  I had the crazy idea that we should design our platform as a series of loosely-connected "micro-services" with an HTML5-based user-facing frontend; messages would be passed as   They was almost a mutiny where they tried to have me committed to a psychiatric ward (ok, not really).

2 years ago we were evaluating replacing our use of ZeroMQ with nanomsg/NNG.  The project was predominantly C# so we looking for .Net bindings.  Initially starting with [a fork of csnng](https://github.com/zplus/csnng), but given the critical nature of the component and multiplatform requirements decided to pursue a re-write.  Thus nng.NETCore was born.  After the failure of the company (and thus the project) I've continued to maintain it over the past year.

Switch Expressions:

Not accepted, must be `default:`:
```csharp
switch (whatever)
{
    case _: // NOPE
        break;
    _: // NOPE
        break;
}
```

Bad
```csharp
ushort port = (ushort)System.Net.IPAddress.NetworkToHostOrder(addr.s_family switch
    {
        Native.nng_sockaddr_family.NNG_AF_INET => (short)addr.s_in.sa_port,
        Native.nng_sockaddr_family.NNG_AF_INET6 => (short)addr.s_in6.sa_port,
        // NOPE
        _ => Assert.True(false, addr.s_family.ToString()),
    });
```

You can do `_ => throw new Exception(addr.s_family.ToString()),`.

```csharp
switch (anErr)
{
    default:
    case var ok when ok.IsOk():
        Assert.True(false);
        break;

    case var err when err.IsErr():
        //ok
        break;
};
Assert.False(anErr switch {
    (false, var err, _) => false,
    _ => true,
});
```


```csharp
Assert.True(anOk switch
    {
        (true, ..) => true, // NOPE
        (true, _) => true, // NOPE
        (true, _, _) => true, // OK
    });
```

```csharp
(_, _, ctxb) = ctx; // OK
(var x, var y, ctxb) = ctx; // NOPE
```

```
error CS8184: A deconstruction cannot mix declarations and expressions on the left-hand-side.
```

```csharp
case (false, _, var ok):
    object.Ok = ok;
    return xxx;
case (false, _, instance.Property): // Nope
    return xxx;
```

```
error CS0150: A constant value is expected
```

```csharp
ref var ctx2 = ref ctxb;
(_, _, ctx2) = ctx; // OK
switch {
    case (false, _, ctx2): // Nope
}
```

## Default Interface Implementations

```csharp
public abstract class Socket : ISocket
{
    public nng_socket NngSocket { get; protected set; }

    ISocket IHasSocket.Socket => this;
}
```

The various socket types have specific rules regarding usage.  For example a "publisher" can't receive- it can only send.  I wanted to encode this in the type system, so I gave each socket an explicit type such that correct usage could be enforced at compile-time.  However, a lot of the functionality is identical, as such there's a handful of socket specific functions, and then around 25 they _all_ have in common.  In C++ you might do this with template specialization, C# doesn't support template specialization, but you can fake it with extension methods.  C#8 provides ["default interface implementations"](https://devblogs.microsoft.com/dotnet/default-implementations-in-interfaces/).

Socket contains no data, just member functions that can be shared.


```
Socket.cs(16,54): error CS8707: Target runtime doesn't support 'protected', 'protected internal', or 'private protected' accessibility for a member of an interface. [/Users/j_woltersdorf/projects/nng.NETCore/nng.NETCore/nng.NETCore.csproj]
Socket.cs(18,38): error CS8701: Target runtime doesn't support default interface implementation. [/Users/j_woltersdorf/projects/nng.NETCore/nng.NETCore/nng.NETCore.csproj]
```

The problem?
```xml
<TargetFrameworks>netstandard1.5;netstandard2.0;net461</TargetFrameworks>
```

I assumed the problem was support for the now mostly legacy .Net Framework, but this requires at least .Net Standard 2.1.  However, because this would considerably alter the class hierarchy this isn't the kind of change you can easily hide behind an `#if`.

Ranges

```csharp
foreach (var _ in 0..10)
{ }
```

```
error CS1579: foreach statement cannot operate on variables of type 'Range' because 'Range' does not contain a public instance definition for 'GetEnumerator'
```