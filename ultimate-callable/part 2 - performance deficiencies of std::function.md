# Performance deficiencies of `std::function`

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
