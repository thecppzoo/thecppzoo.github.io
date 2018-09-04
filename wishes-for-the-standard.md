# What I wish would be standard

## Inheriting from primitive data types

Primitive data types are the fastest, best supported types in the language.  Many times I've wished to adapt some of them for particular use cases and had to pay the big cost of wrapping them, pure busy work.

## Why can't you inherit from unions?

## Why can't you have the `reinterpret_cast`s allowed in the language for `constexpr`s?

### By the way, how do you determine the endianness in a pure C++ 11, portable way?

## Obtaining the address of the code that would be called in a virtual function call

In many cases you write polymorphic code in which you know the type of the data won't change.  It is very frequent you may have a container of pointers to objects of the same type, for example, all of the Java containers are exactly like this, collections of references to objects of exactly the same type that descends from `Object`, hence it is desirable to factor out the addresses of the virtual methods that will be called based on the first element.  But C++ does not give you any way to do this, except mokeying with the vtable implementation of your ABI.  For example,

```c++
#include <tuple>

struct Polymorphic {
    virtual int p1();
    virtual int p2();

    int callBoth() {
        return p1() + p2();
    }

    // implementations
    static int i1(Polymorphic &);
    static int i2(Polymorphic &);

    virtual std::tuple<
        int (*)(Polymorphic &),
        int (*)(Polymorphic &)
    > implementations() {
        return { i1, i2 };
    }
};

int doubleDynamic(Polymorphic &p) { return p.p1() + p.p2(); }
int alsoDoubleDynamic(Polymorphic &p) { return p.callBoth(); }

#include <vector>

int wholeArray(std::vector<Polymorphic *> &collection) {
    auto a = 0;
    for(auto p: collection) { a += p->p1() + p->p2(); }
    return a;
}

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

Although according to my understanding of the language in `alsoDoubleDynamic` to change the type of a polymorphic object referred to by `p` would invalidate `p`, neither GCC nor Clang take advantage of this to factor out the dynamic dispatch, they both generate dynamic dispatch for `p1()` and `p2()`, as indicated by `mov rax, qword ptr [rbx]`, which is to refresh the pointer to the vtable.

In case you are wondering if you could factor out the virtual method addresses by using a pointer to member function, that gets even worse,
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

There is no portable way to leverage the fact that the programmer knows the objects will have the same type.  The programmer needs to provide access to the vtable as in the code for `implementations` above.

