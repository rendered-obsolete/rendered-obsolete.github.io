---
layout: post
title: Lumberyard EBus- 
tags:
- lumberyard
- gamedev
- rust
- cpp
---

One of the architectural changes Amazon made to CryEngine is the introduction of [EBus (Event Bus)](https://docs.aws.amazon.com/lumberyard/latest/userguide/ebus-intro.html) which is:
> a general-purpose communication system that Lumberyard uses to dispatch notifications and receive requests


[TickBus](https://docs.aws.amazon.com/lumberyard/latest/userguide/component-entity-system-pg-tick-bus.html)
```cpp
#include <AzCore/Component/TickBus.h>

class MyEbusHandlerComponent
    : public Component
    , public AZ::TickBus::Handler
{

private:
    void Activate() override
    {
        // Connect to the bus
        TickBus::Handler::BusConnect();
    }

    void OnTick(float deltaTime, ScriptTimePoint time) override
    {
        // Handle tick event
    }
};

// Send a message on `TickRequestBus`
float frameDeltaTime = 0.f;
TickRequestBus::BroadcastResult(frameDeltaTime, &AZ::TickRequestBus::Events::GetTickDeltaTime);
```

## Down the Template Rabbit Hole

Let's take a look at what's going on by unraveling the innocuous looking:
```cpp
TickBus::Handler::BusConnect();
```

`TickBus` is defined in [Code/Framework/AzCore/AzCore/Component/TickBus.h](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/Component/TickBus.h#L153):
```cpp
/**
* The EBus for tick notification events.
* The events are defined in the AZ::TickEvents class.
*/
typedef AZ::EBus<TickEvents>    TickBus;
```

Ok, so we're working with an EBus.  `EBus` is defined in [Code/Framework/AzCore/AzCore/EBus/EBus.h](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/EBus/EBus.h#L345):
```cpp
template<class Interface, class BusTraits = Interface>
class EBus
    : public BusInternal::EBusImpl<AZ::EBus<Interface, BusTraits>, BusInternal::EBusImplTraits<Interface, BusTraits>, typename BusTraits::BusIdType>
{
    //...
```

Because only one template parameter is specified, both `Interface` and `BusTraits` template parameters are the same type (i.e. `EBus<TickEvents, TickEvents>`).

`EBusImpl` is defined in [Code/Framework/AzCore/AzCore/EBus/BusImpl.h](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/EBus/BusImpl.h#L542):
```cpp
template <class Bus, class Traits>
struct EBusImpl<Bus, Traits, NullBusId>
    : public EventDispatcher<Bus, Traits>
    , public EBusBroadcaster<Bus, Traits>
    , public EBusBroadcastEnumerator<Bus, Traits>
    , public AZStd::Utils::if_c<Traits::EnableEventQueue, EBusBroadcastQueue<Bus, Traits>, EBusNullQueue>::type
{
};

//...

template <class Bus, class Traits>
struct EBusBroadcaster
{
    /**
    * An event handler that can be attached to only one address at a time.
    */
    using Handler = typename Traits::BusesContainer::Handler;
};
```

It inherits from a few things, but the one we care about is `EBusBroadcaster` which defines `Handler`.

So, to summarize what we have figured out so far:
```cpp
//TickBus::Handler::BusConnect();
EBusImplTraits<_, TickEvents>::BusesContainer::Handler::BusConnect();
```

`EBusImplTraits` is defined in [BusImpl.h](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/EBus/BusImpl.h#L94)
```cpp
template <class Interface, class BusTraits>
struct EBusImplTraits
{
    //...

    /**
    * Contains all of the addresses on the EBus.
    */
    using BusesContainer = AZ::Internal::EBusContainer<Interface, Traits>;
```

```cpp
//TickBus::Handler::BusConnect();
//EBusImplTraits<_, TickEvents>::BusesContainer::Handler::BusConnect();
AZ::Internal::EBusContainer<_, TickEvents>::Handler::BusConnect();
```

[Code/Framework/AzCore/AzCore/EBus/Internal/BusContainer.h](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/EBus/Internal/BusContainer.h#L93):
```cpp
// Default impl, used when there are multiple addresses and multiple handlers
template <typename Interface, typename Traits, EBusAddressPolicy addressPolicy = Traits::AddressPolicy, EBusHandlerPolicy handlerPolicy = Traits::HandlerPolicy>
struct EBusContainer
{
public:
    //...

    using Handler = IdHandler<Interface, Traits, ContainerType>;
```

Thing is, `IdHandler` doesn't define `BusConnect()`.  The comment gives you a clue about what's going on- [specialization](https://en.cppreference.com/w/cpp/language/template_specialization)/[partial specialization](https://en.cppreference.com/w/cpp/language/partial_specialization) based on `AddressPolicy` and `HandlerPolicy` from the trait (i.e. `TickRequests`).

`TickEvents` in [TickBus.h](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/Component/TickBus.h#L65):
```cpp
/**
* Interface for AZ::TickBus, which is the EBus that dispatches tick events.
* These tick events are executed on the main game thread. In games, AZ::TickBus
* dispatches ticks even if the application is not in focus. In tools, AZ::TickBus 
* can become inactive when the tool loses focus.
* @note Do not add a mutex to TickEvents. It is unnecessary and typically degrades performance.
*/
class TickEvents
    : public AZ::EBusTraits
{
    //...
```

`EBusTraits` in [EBus.h](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/EBus/EBus.h#L75):
```cpp
struct EBusTraits
{
public:
    /**
        * Defines how many handlers can connect to an address on the EBus
        * and the order in which handlers at each address receive events.
        * For available settings, see AZ::EBusHandlerPolicy.
        * By default, an EBus supports any number of handlers.
        */
    static const EBusHandlerPolicy HandlerPolicy = EBusHandlerPolicy::Multiple;

    /**
        * Defines how many addresses exist on the EBus.
        * For available settings, see AZ::EBusAddressPolicy.
        * By default, an EBus uses a single address.
        */
    static const EBusAddressPolicy AddressPolicy = EBusAddressPolicy::Single;
```

[`EBusContainer<_, _, EBusAddressPolicy::Single, EBusHandlerPolicy::Multiple>`](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/EBus/Internal/BusContainer.h#L1294):
```cpp
// Specialization for single address, multi handler
template <typename Interface, typename Traits, EBusHandlerPolicy handlerPolicy>
struct EBusContainer<Interface, Traits, EBusAddressPolicy::Single, handlerPolicy>
{
public:
    using ContainerType = EBusContainer;
    using IdType = typename Traits::BusIdType;

    //...

    using Handler = NonIdHandler<Interface, Traits, ContainerType>;
```

Now we finally know what `Handler` is:
```cpp
//TickBus::Handler::BusConnect();
//EBusImplTraits<_, TickEvents>::BusesContainer::Handler::BusConnect();
//AZ::Internal::EBusContainer<_, TickEvents>::Handler::BusConnect();
NonIdHandler<_, TickEvents>::BusConnect();
```

Look at definition of `NonIdHandler` in [Code/Framework/AzCore/AzCore/EBus/Internal/Handlers.h](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/EBus/Internal/Handlers.h#L145):
```cpp
/**
* Handler class used for buses with only a single address (no id type).
*/
template <typename Interface, typename Traits, typename ContainerType>
class NonIdHandler
    : public Interface
{
public:
    using BusType = AZ::EBus<Interface, Traits>;
    //...

    void BusConnect();
    void BusDisconnect();
```

Sure enough, there's the declaration for `BusConnect()`.  But, what about the definition?  That's back in [Ebus.h](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/EBus/EBus.h#L1223):
```cpp
template <typename Interface, typename Traits, typename ContainerType>
void NonIdHandler<Interface, Traits, ContainerType>::BusConnect()
{
    typename BusType::Context& context = BusType::GetOrCreateContext();
    AZStd::scoped_lock<decltype(context.m_contextMutex)> contextLock(context.m_contextMutex);
    if (!BusIsConnected())
    {
        typename Traits::BusIdType id;
        m_node = this;
        BusType::ConnectInternal(context, m_node, id);
    }
}
```

`this` is the `AZ::TickBus::Handler`-derived handler instance from which we called `BusConnect()` (e.g. in `Activate()`).  This looks like the end, let's just pinch this off: it grabs a lock and calls `EBus<>::ConnectInternal()` (also in [Ebus.h](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/EBus/EBus.h#L975)), which calls `Connect()` on `EBusContainer` instance member (back in [BusContainer.h](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/EBus/Internal/BusContainer.h#L732)), and adds our handler to the list of handlers.

All that template shenanigans to add a pointer to a list.  Who loves C++?

> Bueller... Bueller... Bueller...


## Rustified

[Bindgen doesn't support](https://rust-lang.github.io/rust-bindgen/cpp.html#unsupported-features) several "advanced" C++ techniques like template functions, partial template specialization, and traits templates.

