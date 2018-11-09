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

The above code is actually 100% legal C++.  The problem is that it is not useful.  `TypeAlias1` and `TypeAlias2` are an artifact of mine to tell GCC the ranges don't overlap, the alignment is fine and the counts are multiple of 16; these two types are like "words" used to convey an idea that can be expressed with other words.  I do not intend users to be creating arrays of `TypeAlias1`, they exist so that whenever the user has (at runtime) proven their array of `float`s satisfy the three properties, that from then on they can be treated as "aligned to 64, in blocks of 64 bytes, non overlapping".  But even if they can guarantee their arrays of floats are otherwise fully compatible, they are not `TypeAlias1` because they were not created like that.  Using the `reinterpret_cast` pointers are a violation of the rules of the language, the rules that I criticize in this article.

This tool, of telling the compiler "from now on treat this memory as if it was of this type" is called **"Type Punning"**.  **It is forbidden**!.  (In case you are thinking about it, type punning through `union` is also forbidden *in C++, not C*).

There are only very few ways to change the type of some memory, one is to do "placement new", but the constructors may have side effects, so converting a type through placement new is not a general solution.  Another is to use `memcpy` or `memmove`, but they have their own restrictions, and in the case of arrays, this would not help.  Also, any of the valid ways to change the type of an object invalidates all previous references and pointers to the original compatible types, except when the conversion is to or from arrays of `char`; I solved the needs for type punning in `AnyContainer` through placement new and this exception, but I figure they increased the work needed (and the tedium) by about 50% of what I would have to do if I had type punning.

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

Krister Walfridsson, a GCC developer, has referred to this subject matter in several occasions, including a discussion about [strict aliasing in C](https://kristerw.blogspot.com/2016/05/type-based-aliasing-in-c.html), that you should read if you need more context about strict aliasing before continuing this article.

Linus Torvalds (the leader of Linux) has severely criticized the rule of strict aliasing in C (which applies to C++):

> The standard simply is not \*important\*, when it is in direct conflict with reality and reliable code
> ...
> a standards paper is just so much toilet paper when it conflicts with reality. It has absolutely _zero_ relevance

From the [linux kernel mailing list](https://lkml.org/lkml/2018/6/5/769)

I am undecided with regards to strict aliasing itself, although I think it is a very bad idea, the reasons I have seen for it, that it enables optimizations, look extremely dangerous to me because to allow the compiler to assume two things don't overlap because they are of different types:

1. seems half useful: most operations work on values of the same type (thus they can alias), so, strict aliasing does not help 
2. it seems the wrong solution: to make up rules in the language to cover for the lack of an actual way to express properties
3. strict aliasing is extremely dangerous, because for legitimate reasons, people reuse the same memory with different types and then the compiler may not refresh a value that changed through a pointer of a different type leading to corruption.
4. It is conceptually a half baked idea.  John Regehr, who has such prominence it can be said he can be authoritative commenting on these subjects, explains it really well in [his blog "The Strict Aliasing Situation is Pretty Bad"](https://blog.regehr.org/archives/1307), potentially a bad idea, period.

I always recommend disable strict aliasing because of #3 above, even though I know how to use strict aliasing for higher performance (just showed it, put the option of `-fno-strict-aliasing` in my code and you'll see).

The reason why type punning is related to strict aliasing is that type punning may lead to strict aliasing violations.  If strict aliasing is a bad idea, then who cares about type punning potentially violating strict aliasing?

Even if you like strict aliasing, it is possible for programmers to type pun without violations of strict aliasing, the code above shows an example of real-life code.

**But for me these two rules are siblings of the same problem: solving the problem of lack of expressiveness the wrong way**, rather than draconian rules the programmers can't control and lead to unnecesssary, tedious, error prone work, we should have instead the capacity to **directly express things**.

## `restrict`

`restrict` means in C the pointers won't overlap.  Why can't we include it as-is in C++? because then we have to answer whether/how `restrict` discriminates the overloads.  In C there are no overloads.  The hardest part, by far, for everyone, of C++ are overloads.

We have already a nightmare with `const`, `volatile`, `constexpr` and even `noexcept`.  There is a concept called the ["abominable function types"](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) where two of those four are involved in, and they get joined by `&` and `&&`.

So, I will be blunt, we can't put `restrict` in C++ as it is in C.

However, it is not just that very important reason.  In C++ we also have references that refer to memory and thus may or may not overlap.  And while we are at that, we ought to remember we also have smart pointers, ranges, array indices, hash map keys that refer to objects that may or may not overlap.  The `restrict` annotation is applied to an object (the pointer) that refers to another object, and is useful not just to the compiler, but it may be useful to the programmer too.  For example, knowing that two sets of map keys have empty intersection mean updating the mapped objects from one set of keys can be done in parallel to the update of the other mapped keys, the order won't matter.

What is needed, then, is the way to express two sets have empty intersection.  Something like this would be a fundamental change to C++.

Remember: the objective is to allow the programmer to discover or prove their objects satisfy guarantees and allow them to express those properties to activate better code paths, typically higher performing.  And we want to avoid copy and pasting<sup>[1: copy and pasting is evil](#copy_and_pasting_is_evil)</sup>.

With regards to expressing properties, invariants, etc., we have the class invariant mechanism, and we will soon have *Concepts*, and the mechanisms are excellent, and lead to great code reuse.  Now, all we need is a way to express in source code, at compilation time, that objects acquired properties (because the program was able to guarantee them at runtime) or lost properties (because things change at runtime), if we can encode, express, arbitrarily complex properties of objects through the type system, the usefulness of changing types for the same objects, i.e. type punning, is only natural, I would say *essential*

A few minutes ago I was informed of the existence of [this paper](http://open-std.org/JTC1/SC22/WG21/docs/papers/2018/p1296r0.pdf), `restrict` as an attribute.  It is a better-than-nothing patch; the problem is that C++ already have too many patches of this level of value, and we take too long to get rid of bad ideas.

<a name="#copy-and-pasting-is-evil">1</a>I need to compile all of my writings on redundant code is very detrimental


