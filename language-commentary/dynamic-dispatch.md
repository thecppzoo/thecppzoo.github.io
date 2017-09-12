A friend recently introduced me to [Dyno](https://github.com/ldionne/dyno)

Interesting project that solves may deficiencies in C++.

I will talk about one in particular, that the double indirection penaly of virtual function calls, that is, dynamic dispatch, is unavoidable in the current standard.  By way of code:

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

Where the loop that begins at `.LLB1_1` reloads the vtable, which makes the call a double indirection.  It is unfortunately a silly mistake in the language that **there is no portable way to "freeze" the target of a virtual dispatch**.  Since the language, for good reasons, allows the same memory to be re-classified, the referenced `Base` could change its type after coming back from the invocation of the virtual function, this forces the compiler to have to reload the vtable.  The error of not allowing the freezing of a virtual dispatch is compounded by the fact that **there isn't a portable way to freeze the vtable itself**:  This is highly technical but important:

The only way to refer to a polymorphic object is through their address, but this address is not exclusive to the caller in the general case, because the object, if it truly is polymorphic, is not "an automatic variable", if it were, it would have a known speific type.  Look at this code:

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

The loop starting at `.L9`.  What is happening here is that by virtue of being a local variable, gcc is applying perhaps [Basic.life#9](http://eel.is/c++draft/basic.life#9):

> If a program ends the lifetime of an object of type T with static, thread, or automatic storage duration and if T has a non-trivial destructor,42 the program must ensure that an object of the original type occupies that same storage location when the implicit destructor call takes place; otherwise the behavior of the program is undefined

However, I think GCC is in the wrong when `Base` has a trivial destructor, [like this](https://gcc.godbolt.org/#g:!((g:!((g:!((h:codeEditor,i:(j:1,source:'struct+Base+%7B%0A++++//virtual+~Base()%3B%0A++++virtual+int+function()%3B%0A%7D%3B%0A%0Aint+usesBase(Base+%26b)+%7B%0A++++int+rv+%3D+0%3B%0A++++for(int+i+%3D+1000%3B+i--%3B+)+%7B%0A++++++++rv+%2B%3D+b.function()%3B%0A++++%7D%0A++++return+rv%3B%0A%7D%0A%0Aint+usesLocal(Base+%26b)+%7B%0A++++Base+local%7Bb%7D%3B%0A++++return+usesBase(local)%3B%0A%7D'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0'),(h:output,i:(compiler:1,editor:1),l:'5',n:'0',o:'%231+with+x86-64+gcc+7.2',t:'0')),k:55.122713074520306,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:g72,filters:(___0:(),b:'0',commentOnly:'0',directives:'0',intel:'0',jquery:'3.2.1',length:1,prevObject:(___0:(),length:1,prevObject:(___0:(jQuery321076281124302046211:(display:'')),length:1))),options:'+-O2+-std%3Dc%2B%2B11+-Wall',source:1,wantOptInfo:'1'),l:'5',n:'0',o:'x86-64+gcc+7.2+(Editor+%231,+Compiler+%231)',t:'0')),k:44.877286925479694,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4)

In summary, because of a language deficiency, C++ does not let you:
1. Convert the target of dynamic dispatch into a local variable
2. Convert the vtable into a local variable
    1. Furthermore, GCC may be wrong at converting the vtable into a local variable when the programmer does not want it
