# Type-erasing callables for ultimate performance part 1

The consumption of events is a frequently important software engineering task.  I come from a background on automated trading, in which trading strategies and execution algorithms need to process financial exchange orders as soon as posible, there are literally fortunes to be made and lost on processing these things fast, and I got results on how to consume events the fastest.  This is what I will be describing in this series.

Every aspect of how to get performance known to me will be covered, that is what I mean by "ultimate".

The advice discussed in this series is a natural continuation to [the work I presented at CPPCon 2016](https://www.youtube.com/watch?v=z6fo90R8q5U), where I explain how to convert the specification of financial exchange messages into events with maximal performance, now, let us talk about how to process events.

## Preliminaries

Before we unleash all of the powers of C++, let us start with the simple and certain.  In terms of performance, the cheapest mechanism to subscribe/publish events is through a function pointer for a callback, and user data that will be forwarded.  For example:

```c++
/// Represents the events of interest, for example, a financial market exchange message
struct Event;

/// An slot allows a receiver of events to manage its subscriptions
struct Slot;

/// Event callbacks return a \c Continuation value to indicate to the publisher
/// information about the execution of the callback
struct Continuation;

struct EventPublisher {
    /// A callback receives the event and the subscriber-supplied data,
    /// the user provided data will be forwarded to the receiver along with the event
    Slot subscribe(Continuation (*callback)(void *, Event), void *userData);

    // ...
}
```

A single-thread implementation of a publisher may be as simple as this:

```c++

struct Continuation {
    /// A conversion to true means in this example to stop publishing
    operator bool() const;

    /// In this example, a Continuation can be built from a boolean
    Continuation(bool);

    // ...
};

void EventPublisher::publish(Event e) {
    for(auto receiver: callbacks_) {
        if(receiver.callback_(receiver.data_, e)) { break; }
    }
}
```

From the perspective of the user, they may have something like this:

```c++
struct Strategy {
    void process(Event);

    /// Notification is the callback
    static Continuation notification(void *strategyPtr, Event e) {
        static_cast<Strategy *>(strategyPtr)->process(e);
        return false;
    }

    // ...
}
```

An example of a subscription:

```c++
publisher.subscribe(&strategy, Strategy::notification);
```

The code that receives an event may need to know for example what is the strategy, that is the role of the parameter `userData` in the subscription.  For generality we are using type erasure of the receiver as a `void *`, there are many choices for type erasure, which [I explain in detail on the design of my implementation of `any`](https://github.com/thecppzoo/zoo/blob/master/design/AnyContainer.md#alternatives-to-any) that are superior to using `void *`, however, what is desired here is not just to type erase the receiver, but also to take into account that the receiver has an entry point, a function call signature to receive the events.

We now have a baseline of performance.  Let us see how close we can keep to this ideal while improving every aspect of our event consumption infrastructure.

### Jargon:  Trampolines, compression and release

One bit of jargon:  I will refer to activation of a callback as *jumping into a trampoline*, because I see two phases: the *compression* of the trampoline, and the *release*.  The *compression* means the code that finds what it is going to do next, getting the jump address in the user supplied callback.  The *release* is the function call into the user callback.

### Design of the argument list for event consuming callbacks

To warm up, let us discuss the choices we have with regards to the order of the arguments in the callback signatures.

If you have the freedom to design the interface for event consuming code, take into account that most consumers of events require explicitly a reference to the subscriber or receiver that is receiving the event.  In the ongoing example, a trading strategy wants to know which strategy instance is receiving the event.  If this is not explicit in the event dispatching, then the recipient of events must somehow find out which is the instance, which in most cases is slower than making it explicit.  This is why in the design above there is a proviso for user data as a `void *` callback parameter.

The user data may not be necessary, so, what is the cost of designing event interfaces with it?:

1. All subscriptions must have the reference to the user data
2. One extra parameter in the trampoline calls

For that cost the APIs acquire insurance against user-code having to find out the execution context *at each event*, *for every subscriber*, and the user can't help in any way because the APIs don't provide a hook.

#### User data must be a callback argument

I make the assumption of the user data being a callback argument because the other choices are strictly inferior:

1. If the reference to user data is wrapped into the event itself, then every callback call requires changing the event.  The event may be binary data that the API can't manipulate, for example, in the material I presented at CPPCon, the CME MPD3 messages are converted to events as-is, there is absolutely no conversion, simply a zero-performance cost of interpreting the binary data as events, but this means the event can't be changed.  Which is a good idea, anyway.
2. There is also the choice of the event API wrapping the user data into API structures accessible through the event callback, which I think is a design mistake.

The problems with the second choice are multiple, we will some of these in greater detail, but a short list:

1. It requires changing API structures for each subscriber notification.
1. It is one extra indirection.  Presumably the user data is important, having an intermediate step of dereferencing the event object for user-code to get to its data seems less performing than having it in the function call arguments.  If you think that there is no real expense here because the consumption of events may want to refer to the API implementation object that is receiving the event to query for anything API specific, then the following choice seems superior:
    1. Rather than the API containing in its own data structures a link to the user data, it must be superior to have it the other way around, **the user data, if the user wants it, to have a link to the API data** structure.  Because:
        1. The API structure associated to a subscriber in principle does not change across events
        1. Allows the user choices on how to reference the API structures, including not referring to them
        2. The use cases in which object consumption refers to the receiver APIs are in practice far less frequent and less critical than purely consuming the event data, therefore, when consuming events,
            1. Access to the user data should be prioritized
            2. Access to event API infrastructure objects may be de-prioritized
    3. Frequently, accessing the API receivers occur as a result of unexpected situations.  This suggests that there should be protocol status changes, error, warning events, rather than the receivers having to interpret normal event data and inspecting API objects for anomalous conditions.
        1. This point is motivation for work in this series:  A good event processing library makes it natural for users to define their own events.  Imagine an automated trading system in which several strategies are working with the same market data events, furthermore, let us say the strategies are cooperating.  If one strategy detects an error that is relevant to other strategies, the best choice seems to be that it generates an event, to be consumed by the other strategies or subscribers to events that represent errors.  This is a more fine-grained approach to handling errors that does not require the normally very expensive mechanism of exceptions.
    4. It prevents the entanglement or coupling between event API and user structures
3. The event consumption APIs, especially the callback signatures, must be application centric, or that they should provide directly the data most relevant to the event consumers, not the data relevant to the event implementations.

If the user data reference is provided explicitly as an argument in the callback, then there is the question of where to put in the argument list, as the first parameter or the last.  There is a performance difference.

Let us exaggerate, to illustrate the point, and say that the events are described by very many parameters, and implement a consumer of events for both callback design choices of as-first-argument or last.  By the way, the following code contains an antipattern I will explain right afterward:

```c++
using byte = unsigned char;

struct MarketDataReceiver {
   // Note: making the processing an instance member function is an antipattern
    void processOrder(
        const char *buffer,
        byte channel,
        short sequenceNumber,
        int instrumentId,
        long nanoseconds
    );

    static void callbackFirst(
        void *mdr,
        const char *buffer,
        byte channel,
        short sequenceNumber,
        int instrumentId,
        long nanoseconds
    ) {
        static_cast<MarketDataReceiver *>(mdr)->
            processOrder(
                buffer, channel, sequenceNumber, instrumentId, nanoseconds
            );
    }

    static void callbackLast(
        const char *buffer,
        byte channel,
        short sequenceNumber,
        int instrumentId,
        long nanoseconds,
        void *mdr
    ) {
        static_cast<MarketDataReceiver *>(mdr)->
            processOrder(
                buffer, channel, sequenceNumber, instrumentId, nanoseconds
            );
    }


    // ...
};

auto byFirst = MarketDataReceiver::callbackFirst;
auto byLast = MarketDataReceiver::callbackLast;
```

The resulting assembler is very indicative of the difference, first Clang:
(you can check [this code in the compiler explorer](https://godbolt.org/z/hxkQmj))

```assembly
MarketDataReceiver::callbackFirst(void*, char const*, unsigned char, short, int, long): # @MarketDataReceiver::callbackFirst(void*, char const*, unsigned char, short, int, long)
        jmp     MarketDataReceiver::processOrder(char const*, unsigned char, short, int, long) # TAILCALL
MarketDataReceiver::callbackLast(char const*, unsigned char, short, int, long, void*): # @MarketDataReceiver::callbackLast(char const*, unsigned char, short, int, long, void*)
        mov     r10, r8
        mov     eax, ecx
        mov     ecx, edx
        mov     edx, esi
        mov     rsi, rdi
        mov     rdi, r9
        mov     r8d, eax
        mov     r9, r10
        jmp     MarketDataReceiver::processOrder(char const*, unsigned char, short, int, long) # TAILCALL
byFirst:
        .quad   MarketDataReceiver::callbackFirst(void*, char const*, unsigned char, short, int, long)

byLast:
        .quad   MarketDataReceiver::callbackLast(char const*, unsigned char, short, int, long, void*)
```

What the assembler for those choices say is that if you put the user data first, the user data assumes the role of the `this` implicit parameter in the call for instance functions (methods), the only work needed is a tail call, a jump.  Clearly, the minimum amout of work to call arbitrary code.

If the user data argument is the last parameter, in C++ we have no leeway, `this` will **always** be the first function call parameter, thus **the entire argument list must be rearranged at the point of calling the actual work**.  An optimizing compiler may prevent the reorganization of parameters in user code if the actual work is an inline function, in the example, if `processOrder` would be an inline function, but don't count on users being able to do this, the callbacks may belong to binary-only interfaces not owned by the user.

By the way, for some reason unknown to me, most professionally designed APIs for event consumption make the mistake of putting the user data reference as the *last* argument of callbacks.  The popular LBM API \[[reference](https://kb.informatica.com/proddocs/Product%20Documentation/2/UMP_5.3.3_API_Reference_en.pdf)] makes this mistake, the callbacks have a field they call "clientd" which is a `void *` at the end of the callback argument list.

Now that we have established the principle that *the reference to the user data must be the first argument in event callback design*, we will later see in detail an associated mistake, when discussing the choices for implementations: the implementation data needed in between trampoline compression and release should occur as the **last** argument.  This implementation mistake happens in rather highly performing libraries such as [nano-signal-slot](https://github.com/NoAvailableAlias/nano-signal-slot/blob/e13f25ac84913fd7a69e5523d581525f2a81de6a/nano_function.hpp#L73).

#### Recommendation: do not implement callbacks as instance member functions

One little digression:  `callbackFirst` and `processOrder` do exactly the same things, but they are different.  One is a non-instance function, the other an instance function.

I would like you could set the callback to the address of `processOrder`, which would prevent the totally unnecessary extra indirection, and [GCC has an extension to let you do it](https://gcc.gnu.org/onlinedocs/gcc/Bound-member-functions.html), but alas, not even Clang supports it, *it is not portable*.  I am known to not make a fetish out of portability, but there is a deeper reason for why your API should not support this extension:  The expression for an instance member function, `&TheClass::theMethod` can refer to a non-virtual or a virtual just the same, indistinguishable from the signature, and the trait does not yet exist to query whether a pointer to a member function is virtual.  Theoretically you could implement the trait, [this answer at Stack Overflow gives you the key idea](https://stackoverflow.com/a/44048358), but still your API will reject user-supplied virtual member functions, or to support them you have to degrade performance doing the equivalent of pointer to instance member functions, which won't work, as [I explain in my wishes for the standard](https://github.com/thecppzoo/thecppzoo.github.io/blob/master/wishes-for-the-standard.md#obtaining-the-address-of-the-code-that-would-be-called-in-a-virtual-function-call).  I think this is a case in which *it is legitimate to force the user to wrap their use of polymorphism*, precisely because C++ does not let you bind a virtual function call, there aren't even GCC extensions for this, thus the only way for a user to preserve performance is for them to manually use any option of their choosing to accomplish this binding.  I have run into this annoyance continually all of the years I've been using C++.

Leverage that a non-instance member function that takes as first argument a pointer or a reference to an instance of its class can be safely converted to a signature that takes `void *` as first argument:  `Continuation (*)(YourClass *, Event)` can be safely converted to `Continuation (*)(void *, Event)`, although using `reinterpret_cast`.  But no worries, it is safe because `void *` is safely converted back and forth to a pointer of any data type of the same "const/volatileness", proven by the language allowing you to use `static_cast`.

In the example above, making `processOrder` an instance member function is the antipattern.  It should be an `static` or class member function, or freestanding function, or a non-capturing lambda:

```c++
static void processOrder(
    MarketDataReceiver &,
    const char *buffer,
    byte channel,
    short sequenceNumber,
    int instrumentId,
    long nanoseconds
);
```

This way you can use it directly as the callback, if you don't, even if everything is perfectly inlined, **you will still pay the price of wrapping the instance-function implementation, even if the wrapper is not necessary**, as proven by the example above in which `callbackFirst` is just a fully inlined wrapper around `processOrder`.

If you think using a capturing lambda might help, remember that a capturing lambda won't ever be the receiver (because you won't be able to make the `this` paramater explicit in the parameter list), thus, any way you set the callback you'll have to wrap the lambda, typically with yet another (non capturing) lambda.  The code below implements this idea, and introduces the small buffer optimization:

```c++
#include <utility>
#include <new>

struct Subscription {
    void *userData_;
    int (*callback_)(void *, const char *);

    template<typename Callable>
    // assume the Callable "fits" in the place of a void pointer
    void make(Callable &&c) {
        using D = std::decay_t<Callable>;
        new(&userData_) D{std::forward<Callable>(c)};
        callback_ = [](void *p, const char *e) {
            return (*static_cast<D *>(p))(e);
        };
    }
};

int call(Subscription &s, char *event) {
    return s.callback_(s.userData_, event);
}

struct MarketDataReceiver {
    static int processOrder(
        void *p,
        const char *buffer
    );

    int pO(const char *e) { return processOrder(this, e); }
};

// Jumps directly to the implementation
void set(Subscription &s, MarketDataReceiver &mdr) {
    s.userData_ = &mdr;
    s.callback_ = MarketDataReceiver::processOrder;
}

void makeUsingNonInstanceFunction(Subscription &s, MarketDataReceiver &mdr) {
    auto capture =
        [amdr=&mdr] (const char *p) mutable { return MarketDataReceiver::processOrder(amdr, p); };
    s.make(capture);
}

void makeUsingInstanceFunction(Subscription &s, MarketDataReceiver &mdr) {
    auto capture =
        [amdr=&mdr] (const char *p) mutable { return amdr->pO(p); };
    s.make(capture);
}
```

Every way you try with lambdas, you get the pesky extra jump, not directly to `processOrder`.  [Code here]( https://godbolt.org/z/nTPRUT).  Furthermore, the user data pointer will be the lambda-closure, never the receiver.

### Do not force users to use inheritance

A frequent mistake event APIs designers make is to force the receivers or subscribers to be part of a hierarchy.  In languages with poor expressivity, such as Java, this is a forced choice (it is the only way to achieve type erasure).  Two major objections come to mind:

1. User types may use implementations from inheritance, thus having to inherit from a `Receiver` either leads to multiple inheritance of implementations or boilerplate code of reimplementations.
2. There are many data types that don't allow inheritance, for example lambdas.

### Part 1 conclusion

Ultimate performance requires the event consumption to occur in a function.  Not an instance-member-function, especially not a virtual member function.  However, using instance member functions are not a big deal, they just introduce a jump indirection.

### What's next?

With regards to your subscriptions being able to handle the lifetimes of the user data given to them, I think this is unrelated to the callback function, thus, all of the requirements relate purely to type erasure.  That is why I began by implementing a maximal performance `any` component:  Substitute `void *` for your `any` in the code above.

The C++ standard library already provides the component for type-erased callables, from a while ago, as `std::function`, which even precedes other alternatives for non-callable type erasure such as `any`, `variant`.  `std::function` is a neat component: it lets you use anything that can be called with a particular function signature, the `std::function` can handle the lifetime of the held callable and everything.  The problem is that it is intolerably slow for the critical use case of calling the held callable.  We will see how.  And then, `std::function` does not help you much in any other way.  Pity that the type erasure work needed to implement `std::function` was never available by itself, and eventually [`std::any` came underwhelming in C++ 17, including an important performance error](https://github.com/thecppzoo/zoo/blob/master/design/AnyContainer.md#performance-bug-in-the-specification).

A great API for event consumption should facilitate user defined-events, the composition of events, an algebra of callables, we will hopefully get there.
