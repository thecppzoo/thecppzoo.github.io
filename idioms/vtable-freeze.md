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

Because runtime polymorphism obviously depends on runtime data somehow, the function that will be called, the "target" can't be known at compilation time, thus, will be very expensive on the first call because the processor won't know where to go to next, and the execution pipeline will stall, typically dozens of clock cycles.

Stallings will occur when the branch predictor mispredicts, for example, when you have containers with pointers to polymorphic objects of different types and you traverse the container in such a way that the type of the next object is not predicted well.

1. You can minimize this cost by making your design in such a way that traversing the containers change the types the fewest times without otherwise incurring on extra costs.

Not all the mispredictions of indirect jumps (calls) have the same cost: The earliest time the processor can get the actual target, when a misprediction is detected, the less it has to stall.

2. Try to calculate the target ahead of the use, as for example converting the dynamic dispatch target into a function pointer, or factoring out of loops obtaining the target.

The branch predictors work by maintaining the equivalent of a hash table from program counter addresses (instruction pointers) where indirect jumps occurs (or calls) to targets.  However, these tables are very expensive to manufacture and thus may be ineffective in many unpredictable cases.  For example, the hash table may have too many colisions (other parts of the program, unrelated, end up sharing just by coincidence the same entries in the table and interfering with each other).  This is an aspect in which micro-benchmarks will make indirect jumps (or calls) appear to be much more efficient than in real life, since micro benchmarks will be nearly ideal conditions for branch predictors to work well.  See [Appendix A](#indirect_predictor) for details

One way in which you can increase the likelihood of branch predictor ineffectiveness is by having lots of indirect jumps in the assembler, thus

3. Minimize the indirect calls by transforming conditionals before indirect calls into conditionals that calculate the jump address and have them converge to a single point of indirect calling.

The reasons stated above are why it is important to eliminate the refresh of the vtable, the
```assembly
move rax, qword ptr [<this_ptr>]
```
to then `call qword ptr rax[8*<vtable method index>]` right away, this *maximizes the chances of the branch predictor to fail*

Also, if you carefully observe the assembler before, the compiler may generate a lot of inefficient assembler for pointer to instance member functions.

4. Do not use pointer to member functions at runtime

### Spectre and Meltdown attacks

I am just getting acquainted with the details of Spectre and Meltdown, the attacks related to changing the machine state through speculative execution.

Indirect jumps are another attack vector:

Let us say we have two classes: "SystemObject" and "UserObject".  Methods of "SystemObject" can access privileged memory, while "UserObject"'s can't.  If we "train" the branch predictor we will act on a "UserObject" with user data, afterward we can call the polymorphic method with *system* or privileged data on a SystemObject, but the branch predictor will predict it is a "UserObject" which will speculatively execute the polymorphic version on "UserObject" which may cause side effects that are not completely reverted when the misprediction is detected.

Reasoning from first principles indicates that **shortening the time frame between misprediction and its detection mitigates** the attack while **not incurring on performance penalties**, yet another reason for precomputing the targets, factoring out of loops their calculation.

## Freeze idiom

The only solutions are those in which the targets can be converted to values efficiently handled by the language/compilers, that is, converted to function pointers.  The only three things portably converted to function pointers are: *class* member functions, freestanding functions, and non-capturing lambdas.

Like in the code above, you can put the actual working code into *implementation functions* (again, class member functions, freestanding functions or non-capturing lambdas) as in `Polymorphic::i1`, `Polymorphic::i2`, wrapping them with virtual functions if desired, and providing direct access to the implementations with a virtual function such as in `Polymorphic::implementations`.

Another choice is to altogether forget about virtual functions and implement the equivalent yourself, as Louis Dionne is doing with [Dyno](https://github.com/ldionne/dyno).

## <a name="indirect_predictor">Appendix A:</a> Indirect Predictor:

Jumping to (calling) a runtime-dependent address is called an "indirect jump".  Jon "Hannibal" Stokes explains in [his overview of the Pentium M](https://arstechnica.com/features/2004/02/pentium-m/3/) a little of how this works, which is relevant today because those techniques are the direct ancestors of today's.

Sometimes the details are not available, and they are hard to measure (see what I said about microbenchmarking), thus I take articles such as [this](https://www.realworldtech.com/cpu-perf-analysis/5/) as general indicators.  A more fundamentals-based analysis is this old article, [Limits of Indirect Branch Prediction](http://hoelzle.org/publications/TRCS97-10.pdf) by Karel Driesen and Urs Hölzle at the University of California Santa Barbara.

There are architectures that have the feature of computing the indirect branch target, as [the article on branch prediction at the wikipedia explains](https://en.wikipedia.org/wiki/Branch_predictor#Indirect_branch_predictor).  That is where I got the links from.

### Postscript

I just got excellent articles on indirect branch predictions from the user @Dcoder at the x86 channel of the cpplang Slack workspace [link](https://cpplang.slack.com/archives/C3VR0DMQA/p1538959823000100):
1. [http://www.cs.binghamton.edu/~dima/micro16.pdf](http://www.cs.binghamton.edu/~dima/micro16.pdf)
2. [https://xania.org/201602/haswell-and-ivy-btb](https://xania.org/201602/haswell-and-ivy-btb)
3. [http://www.ece.uah.edu/~milenka/docs/vuam_ispass09.pdf](http://www.ece.uah.edu/~milenka/docs/vuam_ispass09.pdf)
4. [http://www.cs.binghamton.edu/~dima/taco16_branches.pdf](http://www.cs.binghamton.edu/~dima/taco16_branches.pdf)
These links will take me a while to "digest"
