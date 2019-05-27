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
Basically, even specifying `noexcept` in the AMD64 ABI requires the functions to do the equivalent of a "try {...} catch(...) { terminate(); }", adding to the exception tables.

To reduce the size of exception tables we want to mark funcitons as `noexcept` every time it can be proven they are, because every call to a `noexcept(false)` requires an entry in the exception tables; we can see this by creating a local variable of something that has a destructor.  To let the exception propagate, the rules of the language are that the destructors of the local variables will be called:

```c++
void opaquePreamble();
void opaqueEpilog();

struct HasDestructor {
    HasDestructor();
    ~HasDestructor();
};

void fun() {
    HasDestructor hd;
    opaquePreamble();
    opaqueEpilog();
}
```
becomes:
```assembly
fun():                                # @fun()
        push    rbx
        sub     rsp, 16
        lea     rdi, [rsp + 8]
        call    HasDestructor::HasDestructor() [complete object constructor]
        call    opaquePreamble()
        call    opaqueEpilog()
        lea     rdi, [rsp + 8]
        call    HasDestructor::~HasDestructor() [complete object destructor]
        add     rsp, 16
        pop     rbx
        ret
        mov     rbx, rax
        lea     rdi, [rsp + 8]
        call    HasDestructor::~HasDestructor() [complete object destructor]
        mov     rdi, rbx
        call    _Unwind_Resume
```
