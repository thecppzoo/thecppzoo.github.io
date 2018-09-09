# Destructive move

After a few years occasional consideration of the subject of keeping the semantics of move operations as they are versus the option of "destructive move, I have up my mind, destructive move is superior.

As it is frequently the case, C++ is so expressive, and its mechanisms so well performing, that you can disagree with the language and make an alternative like Louis Dionne with "dyno", which offers runtime polymorphism differently to virtual member functions.

Here I will provide an outline of how to use the semantics of destructive move in your libraries.

I was watching [Sean Parent's presentation "Goals for better code"](https://youtu.be/mYrbivnruYw?t=50m51s), when he mentions the current semantics of move are not efficient.  Before move semantics were added to the language, he had already made libraries with the equivalent of what we would call destructive move.

The gist of the idea is that there is no need to make the "moved-from" objects assignable or destructible.  If "moving-from" constitutes a form of destruction, then there is no need to do anything else with the object.  The inefficiency of move currently is that the user of the guts of the moved from object needs to put a destructible and assignable state into the moved-from, for then the normal rules of the language activate the destructor, and perhaps the convenience for the user to assign something to the moved-from.

Because the destructor is always called at terminating the lifetime of an object, to implement destructive move you have no choice **but to make the destructor trivial**, that is, the destructor that does nothing.  This is the first complication: you have no recourse but to do the equivalent of calling the destructor explicitly for types that will have destructive-move semantics.  I think this is also a reason for why move is not destructive in C++:  Normally the rules for destruction of objects are very nice, the "stack order", the last created is the first destroyed.  However, destructive move changes the order of destruction.

You may not be so radical, you can implement a bunch of operations tagged with "unsafe" like Sean Parent, operations what will violate the class invariant, to then require the programmer some way to re-establish the class invariant in the object.

To see the problem, for example, swapping two `unique_ptr`s:

```c++
void swap(uniqueP &p1, uniqueP &p2) {
    uniqueP tmp{std::move(p1)};
    p1 = std::move(p2);
    p2 = std::move(tmp);
}
```

The code for `swap` is what you would do without knowledge of the internals, using just the public interface.

The issue here is that this will be equivalent, more or less, to this:

```c++
void swap(uniqueP &p1, uniqueP &p2) {
    uniqueP tmp = p1.pointer;
    p1.pointer = nullptr; //< this is the inefficiency of move
    p1.pointer = p2.pointer;
    p2.pointer = nullptr;
    p2.pointer = tmp.pointer;
    tmp.pointer = nullptr;
    delete tmp;
}
```

But neither GCC nor Clang do a good job as you can see for this code

```c++
#include <memory>

using uniqueP = std::unique_ptr<long>;

void noInternals(uniqueP &p1, uniqueP &p2) {
    uniqueP tmp{std::move(p1)};
    p1 = std::move(p2);
    p2 = std::move(tmp);
}

void withInternals(uniqueP &p1, uniqueP &p2) {
    std::swap(p1, p2);
}
```

in the [compiler explorer](https://gcc.godbolt.org/z/CERlqc)

Then, **the practical approach is to not have destructive move exactly but get close by creating the operations you want with an "unsafe" tag in the overloads**.  Frequently, as in the example of swapping two `unique_ptr`s, the unsafe states are only ephemeral, they only happen as intermediates within the context of a larger operation that will leave all participant objects in safe states.

I like Sean Parent's ["non-proposal"](https://github.com/sean-parent/sean-parent.github.io/wiki/Non-Proposal-for-Destructive-Move) (?)

