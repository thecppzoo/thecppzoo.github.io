# Support for `noexcept` in trampolines

Recently I have found support for exceptions is a major factor that increases binary size.  Since C++ 17 the `noexcept` specification is part of the function signatures.

First, consider this example:

```c++
void opaquePreamble();
void opaqueEpilog();

void fun() noexcept {
    opaquePreamble();
    opaqueEpilog();
}
```
gets translated to this:
```assembly
fun():                                # @fun()
        push    rax
        call    opaquePreamble()
        call    opaqueEpilog()
        pop     rax
        ret
        mov     rdi, rax
        call    __clang_call_terminate
__clang_call_terminate:                 # @__clang_call_terminate
        push    rax
        call    __cxa_begin_catch
        call    std::terminate()
```


