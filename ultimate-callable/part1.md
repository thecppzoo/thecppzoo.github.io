# Type-erasing callables for ultimate performance part 1.

The consumption of events is a frequently important software engineering task.  I come from a background on automated trading, in which trading strategies and execution algorithms need to process financial exchange orders as soon as posible, there are literally fortunes to be made and lost on processing these things fast, and I got results on how to consume events the fastest.  This is what I will be describing in this series.

## Preliminaries

In terms of performance, the cheapest way to subscribe to events is by providing a function pointer for a callback, and user data that will be forwarded.  For example:

```c++
/// Represents the events of interest, for example, a financial market exchange message
struct Event;

/// An slot allows a receiver of events to manage its subscriptions
struct Slot;

/// Event callbacks return a \c Continuation value to indicate to the publisher
/// information about the execution of the callback
struct Continuation;

struct EventPublisher {
    /// A callback receives the event and the subscriber-supplied data
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

The code that receives an event may need to know for example what is the strategy, that is the role of the parameter `userData` in the subscription.  For generality we are using type erasure of the receiver as a `void *`, there are many choices for type erasure, which [I explain in detail on the design of my implementation of `any`](https://github.com/thecppzoo/zoo/blob/master/design/AnyContainer.md#alternatives-to-any) that are superior to using `void *`, however, what is desired here is not just to type erase the receiver, but also to take into account that the receiver has an entry point, a function call signature to receive the events.

The C++ standard library already provides that facility, from a while ago, as `std::function`, which even precedes other alternatives for type erasure such as `any`, `variant`.  `std::function` is a neat component: it lets you use anything that can be called with a particular function signature.  The problem is that it is intolerably slow for the critical code of calling it:

```c++
#include <functional>

using Event = char *;
using Continuation = bool;

using S = Continuation(Event);
using F = std::function<S>;

struct Subscription {
    void *userData_;
    Continuation (*callback_)(void *, Event);

    Continuation operator()(Event e) {
        return callback_(userData_, e);
    }
};

template<typename Callable>
Continuation call(Callable &c, Event e) {
    return c(e);
}

auto call_subscription = call<Subscription>;
auto call_function = call<F>;

struct Strategy {
    void process(Event);

    /// Notification is the callback
    static Continuation notification(void *strategyPtr, Event e) {
        static_cast<Strategy *>(strategyPtr)->process(e);
        return false;
    }

    // ...
};

#include <new>

void make_subscription(void *where, Strategy *s) {
    new(where) Subscription{s, Strategy::notification};
}

void make_function(void *where, Strategy *s) {
    new(where) F{[=](Event e) { s->process(e); return false; }};
}
```
