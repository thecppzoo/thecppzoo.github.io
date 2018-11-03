# Performance deficiencies of `std::function`

One type-erasure solution available is to use `std::function`.  That is, a component to take care of holding the event processor supplied by the user, regardless of what the type of the event processor is.  Let us compare that solution to the simple solution we showed before, of solving the type erasure needs through a `void *` and a function pointer.  Because what really matters is the invocation (as opposed to the creation/maintenance of the subscriptions), let us focus on the dispatching.  We have this code:

```c++
// Event is the type that represents what the callbacks will process
// Made a "char *" for illustration purposes
using Event = char *;
// Made a boolean for illustration purposes
using Continuation = bool;

using Timestamp = unsigned long;

// Type erasure using void * and a function pointer
struct LeanSubscription {
    void *userData_;
    Continuation (*callback_)(void *, Event, Timestamp);
};

bool dispatch(LeanSubscription &ls, Event e, Timestamp t) {
    return ls.callback_(ls.userData_, e, t);
}

#include <functional>

// Type erasure using std::function
using FatSubscription = std::function<Continuation(Event, Timestamp)>;

bool dispatch(FatSubscription &ls, Event e, Timestamp t) {
    return ls(e, t);
}

template<typename Subscription>
void publishToSubscriptions(
    Subscription *begin, Subscription *end, Event e, Timestamp ts
) {
    while(begin != end) {
        if(dispatch(*begin++, e, ts)) { break; }
    }
}

void lean(LeanSubscription *b, LeanSubscription *end, Event e, Timestamp t) {
    publishToSubscriptions(b, end, e, t);
}

void fat(FatSubscription *b, FatSubscription *end, Event e, Timestamp t) {
    publishToSubscriptions(b, end, e, t);
}
```

We see with the [compiler explorer](https://gcc.godbolt.org/z/y3S7hd) the dispatch of the "lean" option is this:

```assembly
dispatch(LeanSubscription&, char*, unsigned long):
  mov rax, QWORD PTR [rdi+8]
  mov rdi, QWORD PTR [rdi]
  jmp rax
```

In particular, the code "leaves alone" the parameters for the event and the timestamp.

The dispatching of an `std::function` is this:

```assembly
dispatch(std::function<bool (char*, unsigned long)>&, char*, unsigned long):
  sub rsp, 24
  cmp QWORD PTR [rdi+16], 0
  mov QWORD PTR [rsp], rsi
  mov QWORD PTR [rsp+8], rdx
  je .L6
  lea rdx, [rsp+8]
  mov rsi, rsp
  call [QWORD PTR [rdi+24]]
  add rsp, 24
  ret
.L6:
  call std::__throw_bad_function_call()
```

There are so many ineffiencies!

1. The parameters are put into the stack
2. There is a comparison against zero, which corresponds to whether the `std::function` object has a callable target or has been default-constructed, that potentially throws "bad_function_call".
3. The code to throw `bad_function_call`, which only happens if the `std::function` is default constructed, gets inlined at every place the target is invoked

These inefficiences present in libstdc++ are also present in libc++ as I will show shortly.

There is also an important difference, in the case of the lean dispatching, the dispatching shown directly calls user code, however, `std::function` calls a wrapper it generates internally.  Let us inspect the "trampoline release phase" done at `std::function` with stdlibc++ when the target is a function pointer:

```c++
using Event = char *;
using Continuation = bool;
using Timestamp = unsigned long;

#include <functional>

using FatSubscription = std::function<Continuation(Event, Timestamp)>;

FatSubscription build_fat_subscription(
    Continuation(*fp)(Event, Timestamp)
) {
    return {fp};
}
```

The [compiler explorer](https://gcc.godbolt.org/z/utsJKd) tells us the construction does this:

```assembly
build_fat_subscription(bool (*)(char*, unsigned long)): # @build_fat_subscription(bool (*)(char*, unsigned long))
  mov qword ptr [rdi + 16], 0
  test rsi, rsi
  je .LBB0_2
  mov qword ptr [rdi], rsi
  mov qword ptr [rdi + 24], offset std::_Function_handler<bool (char*, unsigned long), bool (*)(char*, unsigned long)>::_M_invoke(std::_Any_data const&, char*&&, unsigned long&&)
  mov qword ptr [rdi + 16], offset std::_Function_base::_Base_manager<bool (*)(char*, unsigned long)>::_M_manager(std::_Any_data&, std::_Any_data const&, std::_Manager_operation)
.LBB0_2:
  mov rax, rdi
  ret
```
 
Because the dispatch calls `[rdi + 24] `, the "release" of the trampoline begins in what is set to the `std::function` object at offset 24: `std::_Function_handler<bool (char*, unsigned long), bool (*)(char*, unsigned long)>::_M_invoke(std::_Any_data const&, char*&&, unsigned long&&)`.  Note the argument list of the function that takes care of the trampoline release makes several mistakes, from the perspective of performance:

1. The first argument, of type `std::_Any_data const &` is not part of the call signature, that is, `std::function` puts its own implementation artifacts in the place of user-provided signature parameters, something we already discussed.  If you have to put artifacts in a function call, **put them at the end!**
2. The arguments are r-value references.  Not forwarding references, but r-value.  That's why the dispatching had to put the parameters on the stack as opposed to pick them from the original call in the registers.

The `std::function` implementation simulates it does not have a compression and release phases by using r-value references in between.  I think this is a mistake I find no benefit for.  Writing to memory is something to be avoided as much as possible, because in cases of heavy memory contention, it wreaks havoc with the performance.  In following articles I will describe my solutions that are either equally expensive as `std::function` for types without trivial copy and move but preserve value-semantics for value-semantic types.

Here is `_M_invoke`:

```assembly
std::_Function_handler<bool (char*, unsigned long), bool (*)(char*, unsigned long)>::_M_invoke(std::_Any_data const&, char*&&, unsigned long&&): # @std::_Function_handler<bool (char*, unsigned long), bool (*)(char*, unsigned long)>::_M_invoke(std::_Any_data const&, char*&&, unsigned long&&)
  mov rax, qword ptr [rdi]
  mov rdi, qword ptr [rsi]
  mov rsi, qword ptr [rdx]
  jmp rax # TAILCALL
```

We see the arguments get picked by the stack, but of course, the order is scrambled.

## Benefits of `std::function`

`std::function` allows us to fully manage automatically the event receiver.  But is that what an event processing framework wants?  I don't think the event processors or receivers, the subscriptions, should participate in resource ownership, actually, I think they should have value semantics.  All of those responsibilities should be delegated to the subscription manager, as opposed to the subscriptions themselves.

However, there are use cases for non-value semantics event processors, for example, debugging components, that do tracing.  Another plausible use case is when the user wants to transfer the event processor to other parts of their code, thus the advanced type-erasure capabilities of `std::function` are necessary.

If you ought to use sophisticated type erasure, **the performance penalties of `std::function` are not inherent**, I will show how to implement flavors of `std::function` much better.

## Conclusion

The performance penalties of are very significant.  Next article I will show how those penalties are for the most part unnecessary.
