# Sloware

This article is about apparently innocent programming techniques that lead to slow code.

I have one example that I want to share, because this programming technique got the great Andrei Alexandrescu, in his presentation ["Fastware"](https://youtu.be/o4-CwDo2zpg?t=18m2s) earlier this year, for maximum irony.  Since I like the challenge of obtaining the ultimate performance, I tought about it and came with a solution that is nearly twice as fast as Alexandrescu's best for sequential inputs and over 5 times faster for random, linearly distributed, inputs.

## Conditional branch misprediction penalties

I noticed something fishy (at 18:02), the casade of unrolled if-thens.  The unrolled code ends executing the same number of conditional branching, therefore, has to pay the same branch misprediction penalties, and I thought this is the dominant factor in the execution time.  This hunch is confirmed by my recent code in [pokerbotic](https://github.com/thecppzoo/pokerbotic), in x86-64 currently a branch misprediction costs about 24 simple instructions, you can fit a few divisions in that time budget so that partially unrolling the loop to cut by four the number of divisions was not going to have a significant effect if the input data is random.  If the input data is sequential, then the processor resources for branch prediction are at their most effective, minimizing the branch prediction penalty and thus the benefit of cutting by 4 the divisions.  But this is not a realistic expectation, optimizations of this kind are programming to the benchmark.

Then I used reduced strength operations, exactly as explained in the presentation, to get very nearly the same performance improvement, but with very simple code.  Then, since `digits10` is combinatorial logic, I used the tried-and-true approach of combining logic with lookup tables to further optimize.  That's how I got at least 1.8 times an improvement on top of Alexandrescu's best and over 5 times better in the random case.

This is the "most naive implementation", which I called [`digits10_naive`](https://github.com/thecppzoo/inprogress/blob/master/src/main.cpp#L7):

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

(in progress)
