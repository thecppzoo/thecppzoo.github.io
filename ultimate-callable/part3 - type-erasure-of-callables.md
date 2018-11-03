# Type erasure of callables, done much better than `std::function`

I have implemented much more flexible type-erasure of callables, in [`AnyCallable`](https://github.com/thecppzoo/zoo/blob/master/inc/zoo/AnyCallable.h), I will explain here why my implementation is much better.

Let us pick the same example we had before, only using [`zoo::function`](https://github.com/thecppzoo/zoo/blob/master/inc/zoo/function.h), which is just the default alias of `AnyCallable`:

```c++
#include <zoo/function.h>

using Event = char *;
using Continuation = bool;
using Timestamp = unsigned long;


// Type erasure using std::function
using ZooSubscription = zoo::function<Continuation(Event, Timestamp)>;

bool dispatch(Event e, Timestamp t, ZooSubscription &ls) {
    return ls(e, t);
}

template<typename Subscription>
void publishToSubscriptions(
    Subscription *begin, Subscription *end, Event e, Timestamp ts
) {
    while(begin != end) {
        if(dispatch(e, ts, *begin++)) { break; }
    }
}

void publishToZoo(
    ZooSubscription *b, ZooSubscription *end, Event e, Timestamp t
) {
    publishToSubscriptions(b, end, e, t);
}

ZooSubscription makeZooSubscription(Continuation(*fp)(Event, Timestamp)) {
    return {fp};
}
```

Note:  I cheated slightly because I changed the order of the dispatch arguments, at the current implementation, the `AnyCallable`, the basis for `zoo::Callable` prefer the trampoline to be passed as the last argument when invoked but this is a choice under revision.

When suitable preprocessed, we can pass it to the [compiler explorer](https://gcc.godbolt.org/z/l8zGgz), and we get these relevant sections (serendipitously Clang generates the object code in an order suitable for the presentation):

```assembly
dispatch(char*, unsigned long, zoo::AnyCallable<zoo::AnyContainer<zoo::RuntimePolymorphicAnyPolicy<8, 8> >, bool (char*, unsigned long)>&): # @dispatch(char*, unsigned long, zoo::AnyCallable<zoo::AnyContainer<zoo::RuntimePolymorphicAnyPolicy<8, 8> >, bool (char*, unsigned long)>&)
  mov rax, qword ptr [rdx + 16]
  jmp rax # TAILCALL
publishToZoo(zoo::AnyCallable<zoo::AnyContainer<zoo::RuntimePolymorphicAnyPolicy<8, 8> >, bool (char*, unsigned long)>*, zoo::AnyCallable<zoo::AnyContainer<zoo::RuntimePolymorphicAnyPolicy<8, 8> >, bool (char*, unsigned long)>*, char*, unsigned long): # @publishToZoo(zoo::AnyCallable<zoo::AnyContainer<zoo::RuntimePolymorphicAnyPolicy<8, 8> >, bool (char*, unsigned long)>*, zoo::AnyCallable<zoo::AnyContainer<zoo::RuntimePolymorphicAnyPolicy<8, 8> >, bool (char*, unsigned long)>*, char*, unsigned long)
  push r15
  push r14
  push r12
  push rbx
  push rax
  mov r14, rcx
  mov r15, rdx
  mov r12, rsi
  mov rbx, rdi
  sub r12, rdi
.LBB1_1: # =>This Inner Loop Header: Depth=1
  test r12, r12
  je .LBB1_3
  mov rdi, r15
  mov rsi, r14
  mov rdx, rbx
  call qword ptr [rbx + 16]
  lea rbx, [rbx + 24]
  add r12, -24
  test al, al
  je .LBB1_1
.LBB1_3:
  add rsp, 8
  pop rbx
  pop r12
  pop r14
  pop r15
  ret
makeZooSubscription(bool (*)(char*, unsigned long)): # @makeZooSubscription(bool (*)(char*, unsigned long))
  mov qword ptr [rdi], offset vtable for zoo::ValueContainer<8, 8, bool (*)(char*, unsigned long)>+16
  mov qword ptr [rdi + 8], rsi
  mov qword ptr [rdi + 16], offset bool zoo::AnyCallable<zoo::AnyContainer<zoo::RuntimePolymorphicAnyPolicy<8, 8> >, bool (char*, unsigned long)>::invokeTarget<bool (*)(char*, unsigned long)>(char*, unsigned long, zoo::AnyContainer<zoo::RuntimePolymorphicAnyPolicy<8, 8> >&)
  mov rax, rdi
  ret
bool zoo::AnyCallable<zoo::AnyContainer<zoo::RuntimePolymorphicAnyPolicy<8, 8> >, bool (char*, unsigned long)>::invokeTarget<bool (*)(char*, unsigned long)>(char*, unsigned long, zoo::AnyContainer<zoo::RuntimePolymorphicAnyPolicy<8, 8> >&): # @bool zoo::AnyCallable<zoo::AnyContainer<zoo::RuntimePolymorphicAnyPolicy<8, 8> >, bool (char*, unsigned long)>::invokeTarget<bool (*)(char*, unsigned long)>(char*, unsigned long, zoo::AnyContainer<zoo::RuntimePolymorphicAnyPolicy<8, 8> >&)
  mov rax, qword ptr [rdx + 8]
  jmp rax # TAILCALL
```

In there, `dispatch` shows you the work needed for the trampoline compression, `::invokeTarget` is the trampoline release.  As you can see, this code as none of the defficiencies of `std::function`, yet, it is functionally equivalent.  There is one penalty, though, which is the trampoline compression is one indirect jump, and the trampoline release also involves one indirect jump.  This is because the trampoline release needs to account for the possiblity of *absolutely any callable*, thus it needs to "figure out" what is it that is going to be called, either in `AnyCallable` or `std::function`.  This inherent cost can be eliminated by using one form of undefined behavior that in practice should work for most platforms, including the x86-64 popular ABIs, but this requires its own explanation in a successive article.

The section

```aseembly
.LBB1_1: # =>This Inner Loop Header: Depth=1
  test r12, r12
  je .LBB1_3
  mov rdi, r15
  mov rsi, r14
  mov rdx, rbx
  call qword ptr [rbx + 16]
  lea rbx, [rbx + 24]
  add r12, -24
  test al, al
  je .LBB1_1
```

shows all the work done to dispatch to an array of callables, more than half of the innermost dispatch cycle is actually loop control!, and the dispatch looks nearly identical to dispatch to an array of function pointers.  We saw on the other side, there are minimal costs too.

## More improvements

`AnyCallable` relies on a template argument to perform type erasure.  That could be any template mechanism, which makes it extremely powerful.  Already, the canonical type erasure component in the zoo library allows specifying the "small buffer" size and alignment, thus, you can benefit by doing this:

```c++
using LargeAnyCallable = zoo::AnyCallable<zoo::AnyContainer<32, 8>>;
```

That gives you the opportunity to keep within the small buffer callables up to 32 bytes in size...

## Lessons

Why is this component so superior?

1. Mindful design: Whenever designing this, I identified choices and applied common sense until discovering which option is better.
2. The application of Andrei Alexandrescu's "Policy Based Design".  Templates are wonderfully configurable things, hence, they should have a policy to configure them.  That allows the user to trivially supply their own adaptations, improvements, choices.
    1. It is foolish to implement bad type erasure in `any`, `variant`, `optional`, `function`, `expected` and any other monad to be invented.  The rational choice is to implement a type erasure component of the highest quality, as I did in [`AnyContainer`](https://github.com/thecppzoo/zoo/blob/master/design/AnyContainer.md), to then use it as the basis for type erasure needs.  Look at the code for [`AnyCallable`](https://github.com/thecppzoo/zoo/blob/master/inc/zoo/AnyCallable.h), it is a very thin wrapper of `AnyContainer`, I did not even bother to implement assignment operators, copy, move constructors and the rest of the crapola member functions because I made it so the defaults, implicitly generated, were already exactly right, and this is [tested](https://github.com/thecppzoo/zoo/blob/master/test/AnyCallable.cpp).
3. Sensible defaults.  It is pointless to initialize the compressor to a null pointer (pun intented), the sensible default is to initialize the default constructed `AnyCallable` to a compressor which throws `bad_function_call`.
4. Appreciating value semantics!!!
5. Notice `AnyContainer` does not contain a single runtime branch, except when absolutely mandated by the standard




