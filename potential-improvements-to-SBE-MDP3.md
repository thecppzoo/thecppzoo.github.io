### Note: this is a work in progress, check timestamps

One data encoding format, [SBE (Simple Binary Encoding)](https://mechanical-sympathy.blogspot.com/2014/05/simple-binary-encoding.html), and the [Chicago Mercantile Exchange's (CME) Market Data Platform 3 (MDP3)](https://www.cmegroup.com/confluence/display/EPICSANDBOX/CME+MDP+3.0+Market+Data), based on SBE, are used for trading a significant portion of all of the financial instruments traded in the world; they have also been subject matters I had to work for some time.

I think these two are representative of the things made in the world of software engineering we have to deal with on a regular basis, and provide insight into opportunities to apply C++ advantages, or the missed opportunities.

When I began studying for my job SBE/MDP3, I was expecting a piece of work with the brilliance or excellence readily appreciable in things like Ken Thompson's UTF8, Eric Niebler's Ranges, Stepanov's Hewlett Packard's STL.  What I found was readily underwhelming.

With regards to the reference implementation code, I never liked it one bit (see [below](#real-logic-code-issues)), but I wasn't expecting good C++ code from Java programmers (the designers of SBE/MDP3 work predominantly with Java).

I had a much different view, after evaluating Onyx's, MayStreet's Bellport and Redline's InRush MDP3 implementations I knew you could do [much better](https://github.com/thecppzoo/cppcon2016), and succeeded at many things.  I was trying to apply specific C++ principles and advantages such as:
* using value semantics,
* hoisting to compilation time as much as possible,
* respecting the original specifications data layouts so that the implementation did not have to make conversions or transformations;

All of this was accomplished, which led to an implementation that, to my knowledge, has the best general purpose processor, single thread performance as reported by all of the colleagues who have independently measured it.  But also, while doing this I discovered the techniques were general, provided the ideas and the building blocks to palliate the lack of C++ instrospection so that you could do generic algorithms on arbitrary protocol and data layout specifications; this gives you maximal performance, and if you can stomach the metaprogramming necessary, also minimal development effort for maintenance; I then [communicated this to the public](https://www.youtube.com/watch?v=z6fo90R8q5U), and my ideas, code were incorporated in the [public ideas, code of people unrelated to me](https://www.youtube.com/watch?v=WlhoWjrR41A), despite the fact that I was expressly forbidden by my former employer to make any publicly available improvements to the work already published; that is, I claim to have succeeded at implementing SBE/MDP3 beyond even the products that several companies sell, which puts me in a position to say what I would have liked on the design itself of SBE/MDP3.

## Oversights.

MDP3: One of the tenets of SBE is to prevent alignment issues that detract performance.  However, inexplicably, the MDP3 [packet begins misaligned](https://www.cmegroup.com/confluence/display/EPICSANDBOX/MDP+3.0+-+Binary+Packet+Header), there is a 32-bit data element immediately followed by a 64-bit data item.  This requires you to read the market messages at a 4-byte offset, which complicates things like kernel-bypass hardware/libraries that expect reads to begin at memory regions aligned at comparatively large sizes such page boundaries (4k in x86-64).  Back in 2016 there were other alignment violations.

## More substantive issues

### Arrays of aggregates as opposed to aggregates of arrays

One concept in SBE is that of "Repeating Groups", which reprents arrays of data, such as the "quotes" (bids, asks) that have occurred for a financial instrument.  The repeating groups have a "controller", a data element that says the length in bytes and the number of elements in the repeating group.  But then the repeating groups occur as an array of aggregates, of heterogenous data elements.

On the large scales, as in the database industry, arrays of aggregates have been the norm, the row-oriented storage and handling of the data, because they support more naturally the relational model, or queries that involve many fields (columns) of the rows.  However, the performance of indices and grouping queries (such as `COUNT`) suffer from heterogenous persistance, in general, all processing mechanisms work better with homogeneous data, especially parallelism; this has led to increasing interest in [column-oriented databases](https://en.wikipedia.org/wiki/Column-oriented_DBMS), that is, databases that tend to persist the data as arrays of columns.

In SBE itself, many types which are members of repeating groups have padding to respect inter-element alignment, this is an inefficiency that matters because it adds to the bulk of network messages payload, and raw transmission time is a significant component of network messages latency.

It is a an often repeated recommendation, even in C++, to not have boolean members for types that will exist in arrays.

For small arrays, for example, a single AVX512 instruction can do many operations, such as price scaling, 8 double-precision floating point elements at a time (see code example below), and if the array have counts not multiple of 8, the number of iterations can frequently be adjusted using masks, which have the added benefit of transforming otherwise code dependencies (branching) into data dependencies (table lookups), further improving performance, and this frequently can be accomplished by the compiler, if not, hand-coded assembler can capture these benefits.   We can compare two sets of up to 8 single byte values using just one 64-bit assembler instruction.  I have done many things like this, such as the mechanisms in Pokerbotic that use [SWAR techniques (SIMD Within A Register)](https://github.com/thecppzoo/pokerbotic/blob/master/inc/ep/Poker.h), which is the foundation for cache resilient, or unaffected by cache usage pressure, potentially world record performance mechanisms.  The techniques in Pokerbotic are partially described [here](https://github.com/thecppzoo/pokerbotic/blob/master/design).

That SBE defines repeating groups as arrays of *heterogeneous data elements*, puzzles me, because I have watched many videos by Martin Thompson, one of the authors, on the subject of high performance software in Java, and in them he has repeatedly explained how it is advisable to de-aggregate data arrays into arrays of homogeneous data.  Of course, the root of the problem is that Java does not have any data layout composition mechanism; all you can do in Java is to define specific layouts that you can't compose, because all classes in Java, user defined layouts, imply referential semantics, the exception being the arrays of primitive data types; furthermore, the composition of classes are also referential (classes that contain members of class types in reality contain *references* to the member instance).  This is an issue in many other popular languages too.

In summary, heterogeneous arrays have three unavoidable consequences contrary to performance:

1. Either padding inefficiency or misalignment penalties
2. Preventing the use of parallel computing resources
3. Reduce memory bandwidth efficiency if the desired operations work on only one member of the aggregates.

Let us see one example, let's have an `Aggregate` that contains a `data` member.  Let us say we want to scale the data by a factor.  Furthermore, let us eliminate the need to copy all of the aggregate if all we want is to work with one member.  Finally, let us make the code exactly identical for both heterogeneous and homogeneous data by using a template:

```c++
struct Aggregate {
    long whatever;
    double data;
    char space[16];
};

static_assert(32 == sizeof(Aggregate));

namespace impl {

// template to guarantee exactly the same type of
// code is used
template<
    typename Data,
    typename Scale,
    typename Accessor
>
void scalePrices(
    Data *__restrict results,
    const Data *__restrict inputs,
    Scale scale,
    Accessor accessor
) {
    constexpr auto q = 1024;
    auto aa = [](auto ptr) {
        auto rv = __builtin_assume_aligned(ptr, 64);
        return reinterpret_cast<Data *>(rv);
    };
    auto destination = aa(results);
    auto source = aa(inputs);
    for(int i = q; i--; ) {
        *accessor(*destination++) = *accessor(*source++) * scale;
    }
}
}

// tight, efficient assembler translation using SIMD
// also, minimal memory bandwidth pressure
void scalePrices(double *to, const double *from, double scale) {
    impl::scalePrices(to, from, scale, [](auto &&v) { return &v; });
}

// ugly assembler
// quadrupled memory bandwidth pressure
// It is also possible "Aggregate" has padding for alignment
void prices(
    Aggregate *to,
    const Aggregate *from,
    double scale
) {
    impl::scalePrices(to, from, scale, [](auto &&v) { return &v.data; });
}
```

The resulting assembler, as the [compiler explorer shows](https://gcc.godbolt.org/z/4XMniU), even disabling loop unrolling, is this:

```assembly
scalePrices(double*, double const*, double): # @scalePrices(double*, double const*, double)
  vbroadcastsd zmm0, xmm0
  xor eax, eax
.LBB0_1: # =>This Inner Loop Header: Depth=1
  vmulpd zmm1, zmm0, zmmword ptr [rsi + 8*rax]
  vmovapd zmmword ptr [rdi + 8*rax], zmm1
  add rax, 8
  cmp rax, 1024
  jne .LBB0_1
  vzeroupper
  ret
.LCPI1_0:
  .zero 8
  .zero 8
  .zero 8
  .zero 8
  .quad 0 # 0x0
  .quad 4 # 0x4
  .quad 8 # 0x8
  .quad 12 # 0xc
.LCPI1_1:
  .quad 0 # 0x0
  .quad 4 # 0x4
  .quad 8 # 0x8
  .quad 12 # 0xc
  .zero 8
  .zero 8
  .zero 8
  .zero 8
.LCPI1_2:
  .quad 8 # 0x8
prices(Aggregate*, Aggregate const*, double): # @prices(Aggregate*, Aggregate const*, double)
  push r14
  push rbx
  vbroadcastsd zmm1, xmm0
  xor eax, eax
  vmovapd zmm2, zmmword ptr [rip + .LCPI1_0] # zmm2 = <u,u,u,u,0,4,8,12>
  vmovapd zmm3, zmmword ptr [rip + .LCPI1_1] # zmm3 = <0,4,8,12,u,u,u,u>
.LBB1_1: # =>This Inner Loop Header: Depth=1
  lea r8, [rdi + rax]
  lea r9, [rdi + rax]
  add r9, 32
  lea r10, [rdi + rax + 64]
  lea r11, [rdi + rax + 96]
  lea r14, [rdi + rax + 128]
  lea rdx, [rdi + rax + 160]
  lea rbx, [rdi + rax + 192]
  lea rcx, [rdi + rax + 224]
  vmovq xmm4, rcx
  vmovq xmm5, rbx
  vpunpcklqdq xmm4, xmm5, xmm4 # xmm4 = xmm5[0],xmm4[0]
  vmovq xmm5, rdx
  vmovq xmm6, r14
  vpunpcklqdq xmm5, xmm6, xmm5 # xmm5 = xmm6[0],xmm5[0]
  vinserti128 ymm4, ymm5, xmm4, 1
  vmovq xmm5, r11
  vmovq xmm6, r10
  vpunpcklqdq xmm5, xmm6, xmm5 # xmm5 = xmm6[0],xmm5[0]
  vmovq xmm6, r9
  vmovq xmm7, r8
  vpunpcklqdq xmm6, xmm7, xmm6 # xmm6 = xmm7[0],xmm6[0]
  vinserti128 ymm5, ymm6, xmm5, 1
  vinserti64x4 zmm4, zmm5, ymm4, 1
  vmovupd zmm5, zmmword ptr [rsi + rax + 8]
  vmovupd zmm6, zmmword ptr [rsi + rax + 136]
  vpermt2pd zmm6, zmm2, zmmword ptr [rsi + rax + 200]
  vpermt2pd zmm5, zmm3, zmmword ptr [rsi + rax + 72]
  vshuff64x2 zmm5, zmm5, zmm6, 228 # zmm5 = zmm5[0,1,2,3],zmm6[4,5,6,7]
  vmulpd zmm5, zmm5, zmm1
  vpaddq zmm4, zmm4, qword ptr [rip + .LCPI1_2]{1to8}
  kxnorw k1, k0, k0
  vscatterqpd zmmword ptr [zmm4] {k1}, zmm5
  add rax, 256
  cmp rax, 32512
  jne .LBB1_1
  xor eax, eax
.LBB1_3: # =>This Inner Loop Header: Depth=1
  vmulsd xmm1, xmm0, qword ptr [rsi + rax + 32520]
  vmovsd qword ptr [rdi + rax + 32520], xmm1
  add rax, 32
  cmp eax, 256
  jne .LBB1_3
  pop rbx
  pop r14
  vzeroupper
  ret
```

The operative part of the homogeneous version is this:

```assembly
.LBB0_1: # =>This Inner Loop Header: Depth=1
  vmulpd zmm1, zmm0, zmmword ptr [rsi + 8*rax]
  vmovapd zmmword ptr [rdi + 8*rax], zmm1
  add rax, 8
  cmp rax, 1024
  jne .LBB0_1
```

Whereas the heterogeneous version has the compiler do inescrutable magic to try to mask away 3/4 of the data so it can use any of its formidable parallel execution resources.

I have chosen an easy to understand example to make a point, in the more representative case of runtime-iteration count, unknown exact alignment, complicated techniques could be used, in C++ code with or without hand-coded assembler, with varying effectiveness; the point is homogeneous data has a rather high ceiling in terms of optimization potential, heterogeneous data is practically hopeless.

If the processing is heterogeneous, or that it requires several members per data element,

* in the case of an heterogeneous encoding, the accesses will require an array base, an index, the offsets to the members and possibly scaling addressing modes, but in the case of x86-64, at least, the "Scale Index Byte" mode won't work, because the straddle between elements is larger than 8, the maximum.
* in the case of homogeneous encoding, the accesses will require several array bases, an index, and almost assured scaling addressing modes, since the straddle between the elements will be up to 8 bytes in most cases...

The compiler can put the array bases in registers.  Thus, **even when the processing is heterogeneous, the homogeneous encoding is advantageous!**

My SBE as part of MDP3 implementation knows all the data sizes at compilation time, but in this regard it is in the minority, the Real Logic implementation converts the XML specification into runtime data, that is, random access indexing operations require actual multiplications!, of course, normally one would iterate over the whole array which gives the opportunity to increment the index by adding the data size; however, if the encoding would be "arrays of aggregates are encoded as aggregate of arrays of primitives", then the scaling addressing modes would be used.

So, in the end the designers of of SBE/MDP3 hit one of the performance pitfalls in most popular programming languages that don't have mechanisms for good data layout specifications and compositions, while they had the opportunity to apply their own recommendations of de-aggregating arrays...

### Doubts about fixed length encoding

Time-based financial exchange market data tends to exibit a lot of similarity.  That is, for example, the timestamps, which in CME MDP3 are 64 bit nanosecond counts, in the overwhelming amount of cases don't vary by more than half of their potential range, price data tends to vary by an epsilon, that could be encoded with a single byte more than 99% of the time, but requires the full 64 bit for fixed-length integer encoding.

As most time-series data, trading data *screams* to be encoded with "delta coding", that is, rather than encode the absolutes, encode the variations, which tend to be small, and then variable-length integer encoding (VLE) can be used for very computationally cheap compression of ratios of 4:1, 6:1.

A drawback of using VLE is that it is stateful.  However, MDP3 is also inherently stateful, it requires the maintenance of the state of something called "level 2 order book", so that is not a big difference.  Another difference is that VLE makes it hard to know ahead of time where the next data element is, but MDP3 as it has always been has the length in bytes of all of the repeating groups, thus, VLE would not introduce extra complications decoding inter-repeating group data.

Thus, if I were CME, I would have requested an encoding which leverages the properties of the actual exchange data, that is, an encoding in which variable length encoding features prominently.

It is public knowledge that one of my work responsibilities was the persistence, archival of market data and that we used VLE delta compression.  I am not allowed to divulge the details of the techniques used, nor their results, but in general there are multiple examples of successful usage of delta-coding with VLE; this is because the compression ratio is so high, and the computations required to compress/decompress so cheap, the reduced data transmission times more than compensates the additional computing latencies.

One of my precursors working on VLE delta coding is Alexander Stepanov himself!, the very author of the STL!, his collaborators and him [tackled the needs for Posting Lists using VLE](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.363.6298&rep=rep1&type=pdf), and for maximal performance of VLE, they illustrated the techniques using SIMD of x86-64.

I have bias toward reducing latency, VLE delta coding is not obviously much better than fixed length encoding to reduce latencies, but for throughput, VLE delta coding makes a **huge advantage**, yet another missed opportunity.

### To be continued 

## Notes:

#### <a name="real-logic-code-issues">Issues in the Real Logic code for SBE:</a>
In general, this is yet another "C with classes" initiative that never uses the true advantages of C++.
In my opinion, it has a strong "feel" of Java, with its unavoidables:

1. Referential semantics, which are strongly required because of
2. class/interface hierarchies,
3. poor data composition.
4. On top of that, they also use type erasure with
    1. notoriously underperforming components such as `std::function`,
    2. the most fundamental meta-language (the language used to define specifications), `Token`, represents type erasure driven by id fields that require profuse, hard to predict branching at runtime
5. Bad "idiomatic" C++, such as passing vectors by shared pointer rather than simply as references...

You can see their [message decoder interface](https://github.com/real-logic/simple-binary-encoding/blob/master/sbe-tool/src/main/cpp/otf/OtfMessageDecoder.h) for an idea
