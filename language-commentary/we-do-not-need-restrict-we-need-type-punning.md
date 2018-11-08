# We don't need `restrict`, we need type punning!

C++ has a particularly powerful type system.  You can use it to tell the compiler quite a lot of things.  For example, that something is aligned: just use `alignas`.  You can also specify that two ranges do not overlap, refer to the two ranges as ranges of two different types; because there is *strict aliasing*, it is a violation to refer to the same memory through pointers of different types.

Look at how you can use that for practical advantage:

```c++
struct AlignedFloat {
    constexpr static auto count = 64/sizeof(float);
    alignas(64) float f[count];

    template<typename Operator>
    auto operate(float operand, Operator operation) {
        for(int i = 0; i < count; ++i) {
            operation(f[i], operand);
        }
    }

    template<typename Operator>
    auto operate(AlignedFloat operand, Operator operation) {
        auto other = operand.f;
        for(int i = 0; i < count; ++i) {
            operation(f[i], other[i]);
        }
    }
};

struct TypeAlias1 {
    AlignedFloat af[1];
};

struct TypeAlias2: TypeAlias1 {};

struct AddInPlace {
    template<typename T>
    T &operator()(T &dest, T source) const { return dest += source; }
};

/// The language says the memory accessed through pointers
/// to different types must be different.  GCC uses this
/// to assume the ranges through b1 to e1 and b2 to e1 - b1 + b2
/// do not overlap.
/// Furthermore, the "TypeAlias" are aligned to 64 bytes
inline void addAVX(
    TypeAlias1 *b1,
    TypeAlias1 *e1,
    const TypeAlias2 *b2
) {
    static_assert(64 == alignof(TypeAlias1), "");
    auto
        current = b1->af,
        end = e1->af;
    auto other = b2->af;
    while(current != end) {
        current->operate(*other, AddInPlace{});
        ++current;
        ++other;
    }
}

void addThroughTypePunningUndefinedBehavior(float *b1, float *e1, const float *b2) {
    // Even if the programmer can guarantee
    // b1, e1, b2 are such that they are fully compatible
    // with addAVX, if b1, e2 were not constructed as
    // TypeAlias1 or if b2 was not constructed as TypeAlias2,
    // then one of the reinterpret_cast below are undefined
    // behavior
    addAVX(
        reinterpret_cast<TypeAlias1 *>(b1),
        reinterpret_cast<TypeAlias1 *>(e1),
        reinterpret_cast<const TypeAlias2 *>(b2)
    );
}

void addVectorsExpensive(float *b1, float *e1, const float *b2) {
    while(b1 != e1) {
        *b1++ += *b2;
    }
}
```

The [compiler explorer shows us what GCC 8.2 generates](https://godbolt.org/z/TSFvpR) for `addThroughTypePunningUndefinedBehavior`:

```assembly
addThroughTypePunningUndefinedBehavior(float*, float*, float const*):
        cmp     rdi, rsi
        je      .L5
        sub     rsi, 64
        xor     eax, eax
        xor     ecx, ecx
        sub     rsi, rdi
        shr     rsi, 6
        add     rsi, 1
.L3:
        vmovaps zmm1, ZMMWORD PTR [rdi+rax]
        vaddps  zmm0, zmm1, ZMMWORD PTR [rdx+rax]
        add     rcx, 1
        vmovaps ZMMWORD PTR [rdi+rax], zmm0
        add     rax, 64
        cmp     rcx, rsi
        jb      .L3
        vzeroupper
.L5:
        ret
```

This optimal or very nearly optimal assembler, the compiler automatically vectorizes the adding to gulp 16 floats at a time, there is no branching except to the minimal to control the loop, and best of all, nothing weird.

For comparison's sake I put the function `addVectorsExpensive` that gets bloated horribly because obviously, the difference between processing 16 floats per cycle is 16 times faster than one per cycle, so, the compiler tries its best to find the way to vectorize, needs to account for overlapping ranges, alignment issues, and iteration counts not multiples of 16.  I won't paste the assembler here, there is no point.

## Type punning, please!

The above code is actually 100% legal C++.  The problem is that it is not useful.  `TypeAlias1` and `TypeAlias2` are an artifact of mine to tell GCC the ranges don't overlap, the alignment is fine and the counts are multiple of 16, the users should not be creating arrays of `TypeAlias1`.  But even if they can guarantee their arrays of floats are otherwise fully compatible, they are not `TypeAlias1` because they were not created like that.  The `reinterpret_cast` are a violation of the rules of the language, rules that I criticize.  This tool, of telling the compiler "from now on treat this memory as if it was of this type" is called **"Type Punning"**.  **It is forbidden**!.  (In case you are thinking about it, type punning through `union` is also forbidden *in C++, not C*).

There are only very few ways to change the type of some memory, one is to do "placement new", but the constructors may have side effects, so converting a type through placement new is not a general solution.  Another is to use `memcpy` or `memmove`, but they have their own restrictions, and in the case of arrays, this would not help.  Also, any of the valid ways to change the type of an object invalidates all previous references and pointers to the original compatible types, except when the conversion is to or from arrays of `char`; I solved the needs for type punning through placement new and this exception when implementing `AnyContainer`, but I figure they increased the work needed (and the tedium) by about 50% of what I would have to do if I had type punning.

It is not just that you could tell things to the compiler about your objects through type punning, as I did above, but that computing resources, CPUs, GPUs, FPGAs, etc., as a matter of course may treat the same bytes as different types.  So, the lack of type punning forces us the programmer to artificially convert the same bytes of one type to another to enable other operations.  But this is complicated, because you want the compiler to "understand" the conversion does nothing.

You can check [Andreas Weis lighning at CPPCon 17](https://youtu.be/_8vMAkCp0Rc?t=339) on how to add an epsilon to a float in memory:

```c++
void add_ulp(float &f) {
   int tmp;
   memcpy(&tmp, &f, sizeof(float));
   ++tmp;
   memcpy(&f, &tmp, sizeof(float));
}
```

The compiler eliminates the creation and copies to and from `tmp`.  But alas, this can not be done in the general case, because for some reason, [`memcpy` -and `memmove`- are not allowed to create objects that are not "Trivially Copyable"](https://en.cppreference.com/w/cpp/string/byte/memcpy).

If you arrange data types according to the endianness of your platform, with type punning then you get lexicographic comparison almost for free, as I explained in [my old blog](https://thecppzoo.blogspot.com/2016/01/achieving-ultimate-date-continued-good.html), if not, then you have to write a conversion to some temporary of the right type and pray for the compiler to "understand" the conversions back and forth do nothing (as in the example above).

Again, the processors are capable of transferring 64 bytes at a time, but even if you have collections of objects smaller than 64 bytes that can be split every 64 bytes, aligned, etc, *there is no way to enable the compiler to use the operations that work on 64 bytes*, because C++ needlessly forbids type punning, that is, **the programmer can't change the way the bytes are interpreted by the compiler**.

## The relationship with strict aliasing

Strict aliasing means except some corner cases, objects of different types have to be in different locations of memory.

Linus Torvalds (the leader of Linux) has severely criticized the rule of strict aliasing in C (which applies to C++):

> The standard simply is not \*important\*, when it is in direct conflict with reality and reliable code
> ...
> a standards paper is just so much toilet paper when it conflicts with reality. It has absolutely _zero_ relevance

From the [linux kernel mailing list](https://lkml.org/lkml/2018/6/5/769)

I am undecided with regards to strict aliasing itself, although I think it is a very bad idea, the reasons I have seen for it, that it enables optimizations, look extremely dangerous to me because to allow the compiler to assume two things don't overlap because they are of different types:

1. seems half useful: most operations work on values of the same type (thus they can alias), so, strict aliasing does not help 
2. it seems the wrong solution: to make up rules in the language to cover for the lack of an actual way to express properties
3. strict aliasing is extremely dangerous, because for legitimate performance reasons, people reuse the same memory with different types and then the compiler may not refresh a value that changed through a pointer of a different type leading to corruption.
4. It is conceptually a half baked idea

I always recommend disable strict aliasing because of #3 above
