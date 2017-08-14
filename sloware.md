# Sloware

This article is about apparently innocent programming techniques that lead to slow code.

I have one example that I want to share, because this programming technique got the great Andrei Alexandrescu, in his presentation ["Fastware"](https://youtu.be/o4-CwDo2zpg?t=18m2s) earlier this year, for maximum irony.  Since I like the challenge of obtaining the ultimate performance, I tought about it and came with a solution that is nearly twice as fast as Alexandrescu's best for sequential inputs and over 5 times faster for random, linearly distributed, inputs.

## Conditional branch misprediction penalties

I noticed something fishy (at 18:02), the cascade of unrolled if-thens.  The unrolled code ends executing the same number of conditional branching, therefore, has to pay the same branch misprediction penalties, and I thought this is the dominant factor in the execution time.  This hunch is confirmed by my recent code in [pokerbotic](https://github.com/thecppzoo/pokerbotic), in x86-64 currently a branch misprediction costs about 24 simple instructions, you can fit a few divisions in that time budget so that partially unrolling the loop to cut by four the number of divisions was not going to have a significant effect if the input data is random.  If the input data is sequential, then the processor resources for branch prediction are at their most effective, minimizing the branch prediction penalty and thus the benefit of cutting by 4 the divisions.  But this is not a realistic expectation, optimizations of this kind are programming to the benchmark.

Then I used reduced strength operations, exactly as explained in the presentation, to get very nearly the same performance improvement, but with very simple code.  Then, since `digits10` is combinatorial logic, I used the tried-and-true approach of combining logic with lookup tables to further optimize.  That's how I got at least 1.8 times an improvement on top of Alexandrescu's best and over 5 times better in the random case.

This is the "most naive implementation", which I called [`digits10_naive`](https://github.com/thecppzoo/inprogress/blob/master/src/main.cpp#L7), the baseline for what we will discuss later:

```c++
unsigned digits10_naive(unsigned long arg) {
    auto rv = 1;
    while(10 <= arg) {
        ++rv;
        arg /= 10;
    }
    return rv;
}
```

## Cheaper instructions

It is true that the compiler converts the division by 10 into a multiplication.  But he failed to appreciate that if the programmer changes the division by multiplication, the compiler will substitute it by addition chains that are faster.  So, a simple, direct improvement, that Alexandrescu did not consider, of using the not-so-strong operation of multiplication instead of division leads to [this](https://github.com/thecppzoo/inprogress/blob/master/src/main.cpp#L28):

```c++
unsigned digits10_mul(unsigned long arg) {
    auto rv = 1;
    auto m = 10l;
    while(m <= arg) {
        ++rv;
        m *= 10;
    }
    return rv;
}
```

That is, simply multiply a comparator by 10 instead of dividing the argument by 10.  This simple improvement is as fast as the partially unrolled loop,

```c++
unsigned digits10_alexandrescu(unsigned long arg) {
    auto rv = 1;
    for(;;) {
        if(arg < 10) return rv;
        if(arg < 100) return rv + 1;
        if(arg < 1000) return rv + 2;
        if(arg < 10000) return rv + 3;
        arg /= 10000;
        rv += 4;
    }
}
```

The [compiler explorer shows](https://godbolt.org/g/Yj9Un3) that in x86-64 multiplying by 10 gets converted to multipliying by 2 and then by 5 through a `lea` operation, two of the instructions that the processor has the resources to execute three per clock.  The technique of loop unrolling to reduce these fake multiplications is so not promising that I did not bother.

Although Alexandrescu does not mention it, further loop unrolling won't help: the processor is dealing with the if-then cascade in parallel with the division of the next iteration, thus the division is not slowing down much, reducing the divisions won't make things faster.

## Lesson about combinatorial functions

A combinatorial function (a funciton whose return value depends only on the inputs and not on the state of execution) should have sparingly few conditional branches, only to deal with the edge cases.  If one wants to make combinatorial functions faster, one should think harder about the logical operations to calculate the return value.  As explained before, conditional branches when mispredicted are very expensive, and with the ever increasing depths of execution pipelines, ever more so, there is always an increasing budget to replace conditional branching by logic.

This is what I do in [my `digits10` function](https://github.com/thecppzoo/inprogress/blob/master/inc/digits10/digits10.h#L55):

```c++
constexpr unsigned digits10(unsigned long arg) {
    if(arg < 10) { return 1; }
    auto l2f = log2floor(arg);
    using Combination = detail::P10<meta::Indices<sizeof(long unsigned)*8 - 1>>; 
    auto cut = arg < Combination::nextPowerOf10(l2f) ? 0 : 1;
    auto base = Combination::ndigits(l2f);
    return cut + base;
}
```

I calculate the floor of the logarithm base 2 of the argument, which in all platforms should be a very fast operation, to then lookup what is the number of digits that correspond to that power of 2, this is done in `ndigits(...)` and a threshold, `nextPowerOf10(...)`, if there is one, at which the number crosses a power of 10, to add one digit more to the result.  For example, for the number 11000 the floor of its log2 is 13, that corresponds to 8192 or 4 digits, but it is higher than the cutoff for numbers higher than 2^13 but smaller than 2^14 which is 10000, thus, the code adds one, and returns the correct 5.

In principle, I could have gotten a humungously large lookup table, but of course, it would make the program excessively slow due to cache misses in the lookups.  I think this is close to an optimal lookup + logic combination.

## Metaprogramming to make lookup tables

I don't bother doing by hand lookup tables, I try hard to not put "magical numbers" in my code.  Metaprogramming lookup tables is complicated, but I got the hang of it and now every time that it makes sense, the metaprogramming of lookup tables saves me effort and gives me performance and code robustness.  This is something I will be explaining in successive articles because I use it all the time, it saves a lot of effort, and I think my explanations will be of value.

[`nextPowerOf10`](https://github.com/thecppzoo/inprogress/blob/master/inc/digits10/digits10.h#L39) is an example of how to make a lookup table:

```c++
template<typename> struct P10;
template<unsigned long... p2s>
struct P10<meta::IndexPack<unsigned long, p2s...>> {
    static constexpr long unsigned nextPowerOf10(unsigned p2) {
        constexpr unsigned long arr[] = {
            pureconstexpr::nextPowerOf10(1ul << p2s)...
        };
        return arr[p2];
    }

    static constexpr unsigned ndigits(unsigned p2) {
        constexpr unsigned arr[] = {
            (1 + pureconstexpr::log10floor(1ul << p2s))...
        };
        return arr[p2];
    }
};
```

Given a template that "tells the code" an argument to fill in the values of the lookup table, the `IndexPack<unsigned long, p2s...>` allows you to do `auto arr[] = { function(p2s)... };`.  Please observe the elipsis is outside the function call, meaning that the function `function` will be called for each member of the parameter pack.  The code *captures* this array as a `constexpr`, meaning that the values are available at compilation time and all!, an expression such as `static_assert(10 == digits10(~0), "");` is perfectly fine (by the way, `0` is an integer that in my x86-64 will be 32 bits, `~0` sets all the 32 bits to one, and is effectively, the largest 32 bit unsigned number, 4 G)

## Benchmarking

(in progress)

