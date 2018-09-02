# Type-erasing callables for ultimate performance part 1.

The consumption of events is a frequently important software engineering task.  I come from a background on automated trading, in which trading strategies and execution algorithms need to process financial exchange orders as soon as posible, there are literally fortunes to be made and lost on processing these things fast, and I got results on how to consume events the fastest.  This is what I will be describing in this series.

The cheapest way to subscribe to events is by providing a function pointer for a callback, and user data that will be forwarded.  For example:

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
    /// the callback indicates there was an error by returning true, false to indicate all successful
    
    Slot subscribe(Continuation (*callback)(Event event, void *userData));

    // ...
}
```

`std::function` is a neat component: it lets you use anything that can be called with a particular function signature.
