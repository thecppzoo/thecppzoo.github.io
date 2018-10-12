A friend recently introduced me to [Dyno](https://github.com/ldionne/dyno)

Interesting project that solves many deficiencies in the core mechanisms of C++ by using other core mechanisms in an user library to provide a better alternative.  I find this capability of C++ unique among programming languages!

In particular, the performance and usability problems of dynamic dispatch in plain vanilla C++ are improved through a library...

I will talk about one of those issues in particular, that the double indirection penalty of virtual function calls, that is, dynamic dispatch, which is unavoidable in the current standard.  By way of code:

```c++
struct Base {
    virtual ~Base();
    virtual int function();
};

int usesBase(Base &b) {
    int rv = 0;
    for(int i = 1000; i--; ) {
        rv += b.function();
    }
    return rv;
}
```

Gets translated to this:

```assembly
usesBase(Base&):                      # @usesBase(Base&)
        push    rbp
        push    r14
        push    rbx
        mov     r14, rdi
        xor     ebx, ebx
        mov     ebp, -1000
.LBB1_1:                                # =>This Inner Loop Header: Depth=1
        mov     rax, qword ptr [r14]
        mov     rdi, r14
        call    qword ptr [rax + 16]
        add     ebx, eax
        inc     ebp
        jne     .LBB1_1
        mov     eax, ebx
        pop     rbx
        pop     r14
        pop     rbp
        ret
```

Where the loop that begins at `.LLB1_1` and ends at `jne.LBB1_1` reloads the vtable, which makes the call a double indirection.  It is unfortunately a silly mistake in the language that **there is no portable way to "freeze" the target of a virtual dispatch**, I mean for the programmer to tell the compiler the type of the object won't change.  The compiler can't prove the type won't change because the language, for good reasons, allows the same memory to be re-classified, the referenced `Base` could change its type after during the invocation of a function opaque to the compiler, this forces the compiler to have to reload the vtable.  The error of not allowing the freezing of a virtual dispatch is compounded by the fact that **there isn't a portable way to freeze the vtable itself**:  This is highly technical but important:

The only way to refer to a polymorphic object is through their address, but this address is not exclusive to the caller in the general case, because the object, if it truly is polymorphic, is not "an automatic variable", if it were, it would have a known specific type.  Look at this code:

```c++
int localBase(Base &b) {
    Base local(b);
    return usesBase(local);
}
```

There is a `Base` copy.  It has a concrete type.  At least GCC would get rid of the reloading of the vtable, while inlining `usesBase`:

```assembly
localBase(Base&):
        push    rbp
        push    rbx
        xor     ebp, ebp
        mov     ebx, 1000
        sub     rsp, 24
        mov     QWORD PTR [rsp+8], OFFSET FLAT:vtable for Base+16
.L9:
        lea     rdi, [rsp+8]
        call    Base::function()
        add     ebp, eax
        sub     ebx, 1
        jne     .L9
        lea     rdi, [rsp+8]
        call    Base::~Base()
        add     rsp, 24
        mov     eax, ebp
        pop     rbx
        pop     rbp
        ret
        lea     rdi, [rsp+8]
        mov     rbx, rax
        call    Base::~Base()
        mov     rdi, rbx
        call    _Unwind_Resume
```

The loop starting at `.L9` and ending in `jne .L9` does not reload the vtable, it `call Base::function()` explicitly.  What is happening here is that by virtue of being a local variable, gcc is applying perhaps [Basic.life#9](http://eel.is/c++draft/basic.life#9):

> If a program ends the lifetime of an object of type T with static, thread, or automatic storage duration and if T has a non-trivial destructor, the program must ensure that an object of the original type occupies that same storage location when the implicit destructor call takes place; otherwise the behavior of the program is undefined

Notice there is a pre-condition, the presence of a non-trivial destructor.  I think GCC is in the wrong when `Base` has a trivial destructor, simply remove the destructor from `Base`:

```c++
struct Base {
    //virtual ~Base();
    virtual int function();
};

int usesBase(Base &b) {
    int rv = 0;
    for(int i = 1000; i--; ) {
        rv += b.function();
    }
    return rv;
}

int usesLocal(Base &b) {
    Base local{b};
    return usesBase(local);
}
```

And the [compiler explorer shows you GCC calls directly `Base::function` while inlining `usesBase` for `usesLocal`, but Clang does not!](https://gcc.godbolt.org/#g:!((g:!((g:!((h:codeEditor,i:(fontScale:0.7464959999999999,j:1,lang:c%2B%2B,source:'struct+Base+%7B%0A++++//virtual+~Base()%3B%0A++++virtual+int+function()%3B%0A%7D%3B%0A%0Aint+usesBase(Base+%26b)+%7B%0A++++int+rv+%3D+0%3B%0A++++for(int+i+%3D+1000%3B+i--%3B+)+%7B%0A++++++++rv+%2B%3D+b.function()%3B%0A++++%7D%0A++++return+rv%3B%0A%7D%0A%0Aint+usesLocal(Base+%26b)+%7B%0A++++Base+local%7Bb%7D%3B%0A++++return+usesBase(local)%3B%0A%7D%0A%0A%23include+%3Ctype_traits%3E%0A%0Astatic_assert(%0A++++std::is_trivially_destructible%3CBase%3E::value,%0A++++%22%22%0A)%3B%0A'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0')),k:32.29198261045396,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:g82,filters:(b:'0',binary:'1',commentOnly:'0',demangle:'0',directives:'0',execute:'1',intel:'0',trim:'1'),lang:c%2B%2B,libs:!(),options:'-O2',source:1),l:'5',n:'0',o:'x86-64+gcc+8.2+(Editor+%231,+Compiler+%232)+C%2B%2B',t:'0')),k:34.37468405621272,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:clang600,filters:(b:'0',binary:'1',commentOnly:'0',demangle:'0',directives:'0',execute:'1',intel:'0',trim:'1'),lang:c%2B%2B,libs:!(),options:'-O2',source:1),l:'5',n:'0',o:'x86-64+clang+6.0.0+(Editor+%231,+Compiler+%231)+C%2B%2B',t:'0')),k:33.33333333333333,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4)

Remember the precondition is that there is a non-trivial destructor that needs to be called on that memory, if this is the case, it makes sense that the memory can not change the object type.  GCC seems to be making an invalid assumption.

For the implementation of performance-critical components I am warming up to the idea of virtual methods, I see myself, you can see instances in this space, as when implementing `any` or ultimate performance callables, that I am fighting against the weirdness, bad ideas of the standard as it relates to virtual method invocation.

In summary, because of a language deficiency, C++ does not let you:
1. Convert the target of dynamic dispatch into a local variable
2. Convert the vtable into a local variable
    1. Furthermore, GCC may be wrong at converting the vtable into a local variable when the programmer does not want it
