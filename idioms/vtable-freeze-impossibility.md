# Freezing the vtable

This article explains the costs of having dynamic dispatch in inner loops, and shows an idiom to make the penalty minimal.

## What is the issue?

In many cases you write polymorphic code in which you know the type of the data won't change while doing an operation.  It is very frequent you may have a container of pointers to objects of the same type, for example, all of the Java containers are exactly like this, collections of references to objects of exactly the same dynamic type that descends from `Object`, hence it is desirable to factor out the addresses of the virtual methods that will be called based on the first element in the container.  But standard C++ does not give you any way to do this, except mokeying with the vtable implementation of your ABI, or using GCC extensions.  For example,

```c++
#include <tuple>

struct Polymorphic {
    // Two polymorphic functions
    virtual int p1();
    virtual int p2();

    // Calls two polymorphic functions.
    // Will the compilers translate this to use the VTable once or twice?
    int callBoth() {
        return p1() + p2();
    }

    // These functions represent instance member functions, they are
    // class-functions that take *this as first argument for compatibility
    // with function pointers.
    static int i1(Polymorphic &);
    static int i2(Polymorphic &);

    // A polymorphic function that returns two implementation functions
    virtual std::tuple<
        int (*)(Polymorphic &),
        int (*)(Polymorphic &)
    > implementations() {
        return { i1, i2 };
    }
};

// Because the function will be a public symbol,
// the compiler will generate the assembler for the expression
int doubleDynamic(Polymorphic &p) { return p.p1() + p.p2(); }
// Shows there is no difference with the above
int alsoDoubleDynamic(Polymorphic &p) { return p.callBoth(); }

#include <vector>

// Precondition that can not be expressed: the dynamic type of the
// pointers will always be of exactly the same derived type of Polymorphic.
// Thus, you can not avoid the invocations of p1 and p2 to be through dynamic dispatch
int wholeArray(std::vector<Polymorphic *> &collection) {
    auto a = 0;
    
    for(auto p: collection) { a += p->p1() + p->p2(); }
    return a;
}

// The precondition of the elements of the same type can now
// be expressed by using the first element as a model for i1, i2
// thus avoiding the cost of vtable lookup in the inner loop
int wholeArraySingleDynamic(std::vector<Polymorphic *> &collection) {
    auto a = 0;
    if(collection.empty()) { return a; }
    auto [i1, i2] = collection.front()->implementations();
    for(auto p: collection) { a += i1(*p) + i2(*p); }
    return a;
}
```

Will be converted to
```assembly
doubleDynamic(Polymorphic&): # @doubleDynamic(Polymorphic&)
  push rbp
  push rbx
  push rax
  mov rbx, rdi
  mov rax, qword ptr [rbx]
  call qword ptr [rax]
  mov ebp, eax
  mov rax, qword ptr [rbx]
  mov rdi, rbx
  call qword ptr [rax + 8]
  add eax, ebp
  add rsp, 8
  pop rbx
  pop rbp
  ret
alsoDoubleDynamic(Polymorphic&): # @alsoDoubleDynamic(Polymorphic&)
  push rbp
  push rbx
  push rax
  mov rbx, rdi
  mov rax, qword ptr [rbx]
  call qword ptr [rax]
  mov ebp, eax
  mov rax, qword ptr [rbx]
  mov rdi, rbx
  call qword ptr [rax + 8]
  add eax, ebp
  add rsp, 8
  pop rbx
  pop rbp
  ret
wholeArray(std::vector<Polymorphic*, std::allocator<Polymorphic*> >&): # @wholeArray(std::vector<Polymorphic*, std::allocator<Polymorphic*> >&)
  push rbp
  push r15
  push r14
  push r12
  push rbx
  mov r15, qword ptr [rdi]
  mov r12, qword ptr [rdi + 8]
  cmp r15, r12
  je .LBB2_1
  xor ebp, ebp
.LBB2_4: # =>This Inner Loop Header: Depth=1
  mov r14, qword ptr [r15]
  mov rax, qword ptr [r14]
  mov rdi, r14
  call qword ptr [rax]
  mov ebx, eax
  mov rax, qword ptr [r14]
  mov rdi, r14
  call qword ptr [rax + 8]
  add ebx, ebp
  add ebx, eax
  add r15, 8
  mov ebp, ebx
  cmp r12, r15
  jne .LBB2_4
  jmp .LBB2_2
.LBB2_1:
  xor ebx, ebx
.LBB2_2:
  mov eax, ebx
  pop rbx
  pop r12
  pop r14
  pop r15
  pop rbp
  ret
wholeArraySingleDynamic(std::vector<Polymorphic*, std::allocator<Polymorphic*> >&): # @wholeArraySingleDynamic(std::vector<Polymorphic*, std::allocator<Polymorphic*> >&)
  push rbp
  push r15
  push r14
  push r12
  push rbx
  sub rsp, 16
  mov rbx, rdi
  mov rax, qword ptr [rbx]
  cmp rax, qword ptr [rbx + 8]
  je .LBB3_5
  mov rsi, qword ptr [rax]
  mov rax, qword ptr [rsi]
  mov rdi, rsp
  call qword ptr [rax + 16]
  mov r15, qword ptr [rbx]
  mov r12, qword ptr [rbx + 8]
  cmp r15, r12
  je .LBB3_5
  xor ebp, ebp
.LBB3_3: # =>This Inner Loop Header: Depth=1
  mov r14, qword ptr [r15]
  mov rdi, r14
  call qword ptr [rsp + 8]
  mov ebx, eax
  mov rdi, r14
  call qword ptr [rsp]
  add ebx, ebp
  add ebx, eax
  add r15, 8
  mov ebp, ebx
  cmp r12, r15
  jne .LBB3_3
  jmp .LBB3_6
.LBB3_5:
  xor ebx, ebx
.LBB3_6:
  mov eax, ebx
  add rsp, 16
  pop rbx
  pop r12
  pop r14
  pop r15
  pop rbp
  ret
```

While executing `p1` as part of `doubleDynamic`, `alsoDoubleDynamic` or `Polymorphic::callBoth`, it may destroy `*this` and build in-place another object.  However, the programmer is *reusing* the pointer (or reference) by next calling `p2` on it.  This activates a condition in the standard, [basic.life#8.2](http://eel.is/c++draft/basic.life#8.2) (my emphasis):

>If, after the lifetime of an object has ended and before the storage which the object occupied is reused or released, a new object is created at the storage location which the original object occupied, a pointer that pointed to the original object, a reference that referred to the original object, or the name of the original object will automatically refer to the new object and, once the lifetime of the new object has started, can be used to manipulate the new object, if:
>
>    (8.1)
>    the storage for the new object exactly overlays the storage location which the original object occupied, and
>    (8.2)
>    **the new object is of the same type as the original object** (ignoring the top-level cv-qualifiers), and

which allows the compiler to deduce the type of the polymorphic object referred to by `p` is exactly the same as it was before invoking `p1`, otherwise the programmer would have written undefined behavior, but neither GCC nor Clang take advantage of this: they both refresh the vtable entry for `p2` before invoking it, as indicated by `mov rax, qword ptr [rbx + 8]` followed by the call on `rax`.

In case you are wondering if you could factor out the virtual method addresses by using a **pointer to member** function, that gets even worse,
```c++
int stillDoubleDynamicAndWorse(
    Polymorphic &p,
    int (Polymorphic::*p1)(),
    int (Polymorphic::*p2)()
) {
    return (p.*p1)() + (p.*p2)();
}
```
gets converted to
```assembly
stillDoubleDynamicAndWorse(Polymorphic&, int (Polymorphic::*)(), int (Polymorphic::*)()): # @stillDoubleDynamicAndWorse(Polymorphic&, int (Polymorphic::*)(), int (Polymorphic::*)())
  push rbp
  push r15
  push r14
  push rbx
  push rax
  mov r15, r8
  mov r14, rcx
  mov rbx, rdi
  add rdx, rbx
  test sil, 1
  je .LBB2_2
  mov rax, qword ptr [rdx]
  mov rsi, qword ptr [rax + rsi - 1]
.LBB2_2:
  mov rdi, rdx
  call rsi
  mov ebp, eax
  add rbx, r15
  test r14b, 1
  je .LBB2_4
  mov rax, qword ptr [rbx]
  mov r14, qword ptr [rax + r14 - 1]
.LBB2_4:
  mov rdi, rbx
  call r14
  add eax, ebp
  add rsp, 8
  pop rbx
  pop r14
  pop r15
  pop rbp
  ret
```
because you now have all the inefficiencies you had before and the extra of the code having to discover the given pointer to member functions require dynamic dispatch.

## GCC extension
The problem is that **there is no way in standard C++ to say "these pointers to member functions only require static dispatch"**.

There is no portable way to leverage the fact that the programmer knows the objects will have the same type.  The programmer needs to provide access to the vtable as in the code for the polymorphic function `implementations` above.

GCC provides an [extension](https://gcc.gnu.org/onlinedocs/gcc/Bound-member-functions.html) to "freeze" the targets of dynamic dispatch by converting them to a normal function pointer:

```c++
auto p2InstanceMemberFunctionPointer = &Polymorphic::p2;
auto p2Target = reinterpret_cast<int (*)(Polymorphic *)>(p->*p2InstanceMemberFunctionPointer);
```

Will compile with a warning that you can turn off (`-Wno-pmf-conversions`).

But **not even Clang supports it**


## Cost

If the dynamic dispatch target is the same, the branch predictor will "learn" it, that is, there will be in the table for jump targets corresponding to the program counter (or instruction pointer) for the indirect call, the address of the target; unless this entry in the table is later clogged by some other code.  In general, if the branch predictor is allowed to do its job, the performance of the call itself will be identical to a static call from then on, however, the steps to find the target address will be executed, these steps are the true penalty that I am trying to avoid factoring out of the loop the "find the target address" part of dynamic dispatch, the AMD64 code for
```assembly
move rax, qword ptr [<this_ptr>]
```
to then `call qword ptr rax[8*<vtable method index>]`

Also, if you carefully observe the assembler before, the compiler may generate a lot of inefficient assembler for pointer to instance member functions.

