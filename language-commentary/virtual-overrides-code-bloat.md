# No access to the vtable mean code bloat

I was reading [Arthur O'Dwyer and Mingxin Wang's paper on object relocation](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1144r0.html) and agreed with their claim that detecting that a move is trivial, or that a move is destructive, presents an opportunity to avoid code duplication.

In my implementation of `any`, `function`, and others, I use "virtual overrides" of the controller object that does the type erased object maintenance, such as copying, moving, destroying, the primitives for efficient swapping.  It bothers me that if I would be able to determine that the code of some operation such as copy or move can be factored out, because programmers have no control over the vtable, such a thing is inexpressible.

Their example is: move is trivial (just copying the bytes to the destination).  Thus, you'd only ever need the functions to copy bytes based on byte counts and alignments.  If your program consists of 1000 different types of size 8 bytes, aligned to 8 bytes, and all of their moves are trivial, you could have a single function for all of them.  But because the vtable does not reuse functions, the compiler will generate 1000 copies of the same function for each type.  Terrible.

I am researching into how to side step this problem.  However, I am convinced at this point the implementation of `AnyContainer` is suboptimal with regards to code efficiency in an inherent way.  It is because not having control of the vtable prevents programmer directed vtable optimization.
