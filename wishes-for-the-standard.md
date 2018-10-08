# What I wish would be standard

## Inheriting from primitive data types

Primitive data types are the fastest, best supported types in the language.  Many times I've wished to adapt some of them for particular use cases and had to pay the big cost of wrapping them, pure busy work.

## Why can't you inherit from unions?

## Why can't you have the `reinterpret_cast`s allowed in the language for `constexpr`s?

### By the way, how do you determine the endianness in a pure C++ 11, portable way?

## Obtaining the address of the code that would be called in a virtual function call

Discussion of this item got promoted to its own article: [vtable freeze](https://github.com/thecppzoo/thecppzoo.github.io/blob/master/idioms/vtable-freeze.md).
