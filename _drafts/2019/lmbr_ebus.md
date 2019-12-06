---
layout: post
title: Lumberyard EBus from Rust
tags:
- lumberyard
- gamedev
- rust
- ffi
- bindgen
series: Rust in Lumberyard
---

One of the architectural changes Amazon made to CryEngine is the introduction of the [EBus (Event Bus)](https://docs.aws.amazon.com/lumberyard/latest/userguide/ebus-intro.html):
> a general-purpose communication system that Lumberyard uses to dispatch notifications and receive requests

Ebuses are a key participant in the [component](https://docs.aws.amazon.com/lumberyard/latest/userguide/component-intro.html) and [gem](https://docs.aws.amazon.com/lumberyard/latest/userguide/gems-system-gems.html) systems and used throughout the engine.  Their indirection and de-coupling make them an ideal candidate for integrating additional languages in Lumberyard development.

For further details about ebus, see:
- [Usage and Examples](https://docs.aws.amazon.com/lumberyard/latest/userguide/ebus-usage-and-examples.html)
- [Event Buses in Depth](https://docs.aws.amazon.com/lumberyard/latest/userguide/ebus-in-depth.html)
- [EBus API Reference](https://docs.aws.amazon.com/lumberyard/latest/apireference/EBus.html)

## EBus Primer

Let's take a look at using the [TickBus](https://docs.aws.amazon.com/lumberyard/latest/userguide/component-entity-system-pg-tick-bus.html):
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

To subscribe to an ebus and receive published events:
1. Derive from `EBus<T>::Handler`
1. Call `EBus<T>::Handler::BusConnect()`

`Broadcast()` and `BroadcastResult()` are used to send messages (requests) to an ebus.

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

That means we now know:
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

Therefore the specialization we're interested in is [`EBusContainer<_, _, EBusAddressPolicy::Single, EBusHandlerPolicy::Multiple>`](https://github.com/aws/lumberyard/blob/f3d7d36f55340b60e8e5b2b68a264cec33276c2e/dev/Code/Framework/AzCore/AzCore/EBus/Internal/BusContainer.h#L1294):
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

Now we finally know what `TickBus::Handler` is:
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

All that template shenanigans to add a pointer to a list.  Good times.

## Rust-ified

All of that was very interesting, but how can we call it from Rust?  [Bindgen doesn't support](https://rust-lang.github.io/rust-bindgen/cpp.html#unsupported-features) several "advanced" C++ techniques like template functions, partial template specialization, and traits templates.

This results in a few problems for us:

1. No way to resolve types
    - As we found above, because of template specialization `TickBus::Handler::BusConnect()` is `NonIdHandler<_, TickEvents>::BusConnect()`.  But Bindgen only generates bindings for the base template:  
    ```rust
    pub type EBusContainer_Handler<Interface> = root::AZ::Internal::IdHandler<Interface>;
    ```
1. There's no functions to bind (link) to
    - Almost everything is defined via templates in header files; mostly inlined and not externally visible symbols

The simplest approach is wrapping the needed routines in "plain" C functions like we did to deal with [Lumberyard Editor `IPlugin`]({% post_url /2019/2019-10-05-lmbr_editor_rust %}):
```cpp
#include <AzCore/Component/TickBus.h>
#include <AzCore/Script/ScriptTimePoint.h>

extern "C" {
	void TickRequestBus_BroadcastResult_GetTimeAtCurrentTick(AZ::ScriptTimePoint& results)
	{
		AZ::TickRequestBus::BroadcastResult(results, &AZ::TickRequestBus::Events::GetTimeAtCurrentTick);
	}
}
```

In Rust:
```rust
extern "C" {
    pub fn TickRequestBus_BroadcastResult_GetTimeAtCurrentTick(results: *mut root::AZ::ScriptTimePoint);
}
```

Simple and effective, albeit tedious.  But we're not done yet.

Look at the definition of `ScriptTimePoint` in `Code\Framework\AzCore\AzCore\Script\ScriptTimePoint.h`:
```cpp
class ScriptTimePoint
{
public:
    AZ_TYPE_INFO(ScriptTimePoint, "{4c0f6ad4-0d4f-4354-ad4a-0c01e948245c}");
    AZ_CLASS_ALLOCATOR(ScriptTimePoint, SystemAllocator, 0);

    ScriptTimePoint()
        : m_timePoint(AZStd::chrono::system_clock::now()) {}
    //...
}
```

And `system_clock::now()` in `Code\Framework\AzCore\AzCore\std\chrono\clocks.h`:
```cpp
class system_clock
{
public:
    //...
    static time_point  now()        { return time_point(duration(AZStd::GetTimeNowMicroSecond())); }
    //...
};
```

All of this is inlined leaving us with no way to correctly initialize `ScriptTimePoint` in Rust.

There's a few different ways to deal with this:  

- If we compile Lumberyard with inlining disabled, we _could_ get bindgen to generate a binding with [generate_inline_functions(true)](https://docs.rs/bindgen/0.51.1/bindgen/struct.Builder.html#method.generate_inline_functions):  
    ```rust
    extern "C" {
        #[link_name = "\u{1}?now@system_clock@chrono@AZStd@@SA?AV?$time_point@Vsystem_clock@chrono@AZStd@@V?$duration@_JV?$ratio@$00$0PECEA@@AZStd@@@23@@23@XZ"]
        pub fn system_clock_now() -> root::AZStd::chrono::system_clock_time_point;
    }
    impl system_clock {
        #[inline]
        pub unsafe fn now() -> root::AZStd::chrono::system_clock_time_point {
            system_clock_now()
        }
    }
    ```

- Wrap the function and export it from C++:
    ```cpp
    extern "C" {
        time_point system_clock_now() {
            return system_clock::now();
        }
    }
    ```
    Then in Rust:
    ```rust
    extern "C" {
        pub fn system_clock_now() -> time_point;
    }
    ```

- Re-implement the helper in Rust:  
    ```rust
    impl AZStd::chrono::system_clock {
        pub fn now() -> AZStd::chrono::system_clock_time_point {
            unsafe {
                let time = AZStd::GetTimeNowMicroSecond();
                let duration = AZStd::chrono::duration::from_duration_rep(time);
                AZStd::chrono::time_point::from_time_point_duration(duration)
            }
        }
    }

    impl<Rep> AZStd::chrono::duration<Rep> {
        pub fn from_duration_rep(val: Rep) -> Self {
            Self { m_rep: val, _phantom_0: PhantomData }
        }
    }

    impl<Duration> AZStd::chrono::time_point<Duration> {
        pub fn from_time_point_duration(val: Duration) -> Self {
            Self { m_d: val, _phantom_0: PhantomData }
        }
    }
    ```

At the moment, some combination of the first two seems like the right choice:  

1. Requires least amount of code to write/debug/maintain
1. If original C++ code is removed/renamed, we'll get a compile time error to update the wrapper
1. Leverage bindgen to automatically generate Rust bindings

## Solution

First, we create wrappers in a C++ static library to access from Rust:
```cpp
// RustAz.h
namespace AZStd {
	namespace chrono {
		system_clock::time_point system_clock_now()
		{
			return system_clock::now();
		}
	}
}
// RustAz.cpp
#include "RustAz.h"
```

`RustAz/wscript` to produce `RustAz.lib`:
```python
def build(bld):
    bld.CryEngineStaticLibrary(
        target          = 'RustAz',
        #...
    )
```


Build, and `dumpbin /symbols <root>BinTemp\win_x64_vs2017_profile\Code\Framework\RustAz\RustAz\RustAz.lib` confirms it is an externally visible symbol defined in the library:
```
287 00000000 SECT113 notype ()    External    | 

?system_clock_now@chrono@AZStd@@YA?AV?$time_point@Vsystem_clock@chrono@AZStd@@V?$duration@_JV?$ratio@$00$0PECEA@@AZStd@@@23@@12@XZ 

(class AZStd::chrono::time_point<class AZStd::chrono::system_clock,class AZStd::chrono::duration<__int64,class AZStd::ratio<1,1000000> > > __cdecl AZStd::chrono::system_clock_now(void))
```

Next, process `RustAz.h` with bindgen to generate the Rust bindings:
```rust
pub mod AZStd {
    pub mod chrono {
        extern "C" {
            #[link_name = "\u{1}?system_clock_now@chrono@AZStd@@YA?AV?$time_point@Vsystem_clock@chrono@AZStd@@V?$duration@_JV?$ratio@$00$0PECEA@@AZStd@@@23@@12@XZ"]
            pub fn system_clock_now() -> root::AZStd::chrono::system_clock_time_point;
        }
    }
}
```

## Ebus-in'

From Rust we can now make a request via the ebus and receive a simple result:
```rust
unsafe {
    use lmbr_sys::root::{AZ, AZStd};
    let mut current_time = AZ::ScriptTimePoint { m_timePoint: AZStd::chrono::system_clock_now() } ;
    lmbr_sys::TickRequestBus_BroadcastResult_GetTimeAtCurrentTick(&mut current_time);
    info!("GetTimeAtCurrentTick {:?}", current_time);
}
```

I must admit, the prospect of having to re-write and manually maintain extensive bindings is daunting and dissuades me from wanting to continue pursuing this.  However, there doesn't seem to be a better option until bindgen gets support for more than the most trivial C++.