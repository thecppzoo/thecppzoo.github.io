# The idiom of the "power set covariant return type" template pattern revisited

I wrote a set of five articles for my blog that cover aspects of the power set covariant return type idiom.  Because of a presentation by Ben Deane at CPPCon 2018 on ["Declarative Style..."](https://cppcon2018.sched.com/event/FnLS/easy-to-use-hard-to-misuse-declarative-style-in-c) (video nor slides yet available) which mentions it, I thought about consolidating the old writings into a self-sufficient article on the subject.

## Partial lists of function call arguments

In my article ["Lists of function call arguments"](https://thecppzoo.blogspot.com/2015/12/lists-of-function-call-arguments.html) I refer to something which is not expressible in C++, lists of function call arguments.  This insuficiency is more glaring when they make function calls error prone.  One example is when conveying to the function a set of boolean configuration flags, a concrete example, let's say a typografic font may have the options of being underlined, bold, italic or strike-through.  This makes sense in C++:

```c++
struct Font {
  // ...
   Font(const std::string &name, int size, bool underlined, bool bold, bool italics, bool strike_through);
```

a constructor that takes a bunch of booleans.  The problem is that a call site

```c++
    auto font = Font{"Times New Roman", 14, false, true, false, true};
```

is notorious for how hard it is to understand it:  What do those `false`s and `true`s mean?  This is error prone.

If the symbol `#` could be used as syntax for an operator to indicate composition of partial argument lists, and `<#>` as a declarator of partial argument lists, the above line could be done this way, assuming `b` is a partial argument list in which "bold" is set, `stt` "strike-through",

```
    <#> bold_and_strike_through = b # stt;
    auto font = Font{"Times New Roman", 14, # bold_and_strike_through};
```

But such syntax does not exist at all.  C++ is deficient with respect to the argument lists, there is syntax only for the position.  Python, for example, at least allows named parameter lists (dictionary in Python's case), but of course, incurs a runtime cost for that.  More generally:

1. Positional parameters are brittle: the function owner can remove and add a parameter and the call sites get invalidated without creating an error because the total number of parameters and the order of types the same
2. Most of the times the order of parameters is arbitrary, there is no objective superior order; thus
    1. There is no practical way to express default values
    2. because it is arbitrary, programmers must remember the arbitrary order,
        1. which is a productivity waste,
        2. forces programmers to remember useless things,
        3. to count positions,
        4. to jump back and forth between declaration and call site
4. It is a wasted opportunity for expressing the rather high cohesion between sets of parameters
5. Since funciton overload resolution is a hugely important feature of C++, at the core of all pattern matching, this issue complicates every C++ programmer's life significantly.

However, C++ has otherwise excellent expressiveness, there are choices to mitigate the problems.

Dear Matthew Wilson, the author of "QM Bites" [Dec 2015](https://accu.org/index.php/journals/2183) explains that we should rather not use boolean arguments.  I don't disagree with the advice, given the practical considerations, my objection to the "Quality Matters Bite" is that the presentation of the problem is well done, but the consideration of the solutions don't include the fundamentals it should.

## Configuration classes

Let us hire C++ expresiveness to help with function call argument list:  In the case of font options, we can use their cohesion to pack them into a `FontFlags` or `FontOptions` type:

```c++
struct FontOptions {
    enum Options: unsigned char {
        UNMODIFIED = 0,
        BOLD = 1,
        ITALIC = 2,
        UNDERLINED = 4,
        STRIKE_THROUGH = 8
    };

    unsigned char value = 0;

    FontOptions() = default;
    FontOptions(const FontOptions &) = default;

    FontOptions unmodified() const { return FontOptions(0); }
    FontOptions bold() const { return value | BOLD; }
    bool is_bold() const { return value & BOLD; }
    FontOptions not_bold() const { return value & ~BOLD; }
    FontOptions italic() const { return value | ITALIC; }
    bool is_italic() const { return value & ITALIC; }
    FontOptions not_italic() const { return value & ~ITALIC; }
    FontOptions underlined() const { return value | UNDERLINED; }
    bool is_underlined() const { return value & UNDERLINED; }
    FontOptions not_underlined() const { return value & ~UNDERLINED; }
    FontOptions strike_through() const { return value | STRIKE_THROUGH; }
    bool is_strike_through() const { return value & STRIKE_THROUGH; }
    FontOptions not_strike_through() const { return value & ~STRIKE_THROUGH; }

    // The usual equality, difference operators, perhaps even the relational
    // not implemented for brevity of exposition

private:
    FontOptions(unsigned v): value(v) {}
};

// ...
struct Font {
    // ...
    Font(const std::string &, int, FontOptions);
```

Allows the incontrovertible easy to understand, write, read, well performing

```c++
    auto font = Font{"Arial", 12, FontOptions{}.italic().bold()};
```

By not requiring any order for the specification of the flags, we have also gained the ability to provide defaults.

But without reflection/introspection nor Herb Sutter's metaclasses, writing `FontOptions` took a lot of work.  We can paliate their lack using macros.  I know, the usefulness of the rest of the discussion is suspect because I am advocating for a paliative for the lack of list of arguments using an idiom that itself requires a paliative... As a team leader I would rather my teams express as much as they can in code, even if such expression requires using a multiplicity of techniques.  **That is the fundamental element of my doctrine for software engineering: Express everything in code**.

I will quote at length my old article, which assumed knowledge of yet another deficiency in C++, that you can't handle programmatically identifiers, discussed in my article [Introspection, preprocessing, ...](https://thecppzoo.blogspot.com/2015/11/introspection-preprocessing-and-boost.html):

As you can see, instead of suffering of problems mentioned, this example:

1. shows how easy it is to understand exactly what are the options set,
2. there is no need to remember the order,
3. introducing new options won’t even necessarily require recompilation, and in particular no code change,
4. and should you want to change the meaning of any of the options, a simple grep of the desired setter will tell you reliably all the call sites so that you change accordingly.

Very probably inside the `Font` implementation you’d want to have a set of options. You could use `FontOptions` exactly as it is inside Font. You could communicate internally the whole set of options using `FontOptions`. It is useful.

Observe that the implementation of `FontOptions` is very repetitive. And incomplete, since the constructors, setters, etc, are not `constexpr` `noexcept`, there is no operator for comparison, etc; it is error prone too, at any moment one could have written `italic` meaning `bold`. These things are best left to something automated. Unfortunately, we begin with a list of identifiers (`bold`, …) and as already explained in the [previous article](https://thecppzoo.blogspot.com/2015/11/introspection-preprocessing-and-boost.html), there is no support in pure C++ for them; we have turned the inexpressible function call argument list into a list of identifiers, thus, as done in the previous article, the next best choice is to use preprocessing:

```c++
#include <boost/preprocessor/seq/for_each_i.hpp>
#include <boost/preprocessor/cat.hpp>

#define PP_SEQ_I_DECLARE_ENUMERATION_VALUE(r, dummy, i, e) e = (1 << i),
#define PP_SEQ_I_FLAG_SET_MEMBER_FUNCTIONS(r, name, i, e)\
    constexpr name e() const noexcept { return value | internal::e; }\
    constexpr name BOOST_PP_CAT(not_, e)() const noexcept { return value & ~internal::e; }\
    constexpr bool BOOST_PP_CAT(is_, e)() const noexcept { return value & internal::e; }

#define BOOLEAN_FLAG_SET(name, flags)\
    class name {\
        struct internal { enum: unsigned char {\
            EMPTY = 0,\
            BOOST_PP_SEQ_FOR_EACH_I(PP_SEQ_I_DECLARE_ENUMERATION_VALUE, ~, flags)\
        }; };\
        unsigned char value = 0;\
        constexpr name(unsigned v) noexcept: value(v) {}\
    public:\
        name() = default;\
        name(const name &) = default;\
        BOOST_PP_SEQ_FOR_EACH_I(PP_SEQ_I_FLAG_SET_MEMBER_FUNCTIONS, name, flags)\
        constexpr bool operator==(name other) const noexcept { return value == other.value; }\
        constexpr bool operator!=(name other) const noexcept { return !(*this == other); }\
    }
```

The use of the macro `BOOLEAN_FLAG_SET` is very easy, for example:

```c++
BOOLEAN_FLAG_SET(FO, (bold)(italic)(underlined)(strike_through));
```

This would declare and define a class `FO` that does the same things as `FontOptions`. I hope you’d agree that declaration and definition takes very little effort and prevents lots of errors. Also, we could improve this so that we also get inserter and extractor operators with minimal effort.

Before explaining the implementation, as a conclusion to this article, I advocate for the following:

1. Because there is no support for list of arguments, avoid them
2. Strive to minimize the number of arguments in your function calls
3. Grouping parameters into a single aggregate tends to capture useful semantics of the application domain that will speed up development and improve quality
4. Pure C++ offers limited support for expressing the groups of parameters identified, typically, the preprocessor may palliate this defficiency.

End of extended quote.

So far, we have a solution for boolean parameters.  We will generalize, but first let us keep it simple by continuing to work on boolean parameters.

Over the years I've developed a convention regarding boost preprocessing.  The macros that will be called by boost preprocessing functions have names with the prefix `PP_`.  If they are called from the [sequence library](https://www.boost.org/doc/libs/1_68_0/libs/preprocessor/doc/headers/seq/) then the prefix is followed by `SEQ_`, and if meant to be called form an indexed sequence macro, `I_`.

## Complete configuration classes, the introduction of the power set covariant return type template pattern idiom

We've lost one thing from the original of passing each flag as a parameter, that each had to be specified.  We should be able to recover that, to continue with the ongoing example, we need a way to reject at compilation time an attempt to create a `Font` with an incompletely specified `FontOptions`.

Conceivably we might use a compile-time value to indicate the flags set, but if we keep an eye toward generalizing this technique for types not boolean, in particular not suitable to be constructed at compilation time, the mix between a compile time value with non-compile time parts seems not promising.  That leaves us with using a calculated type.

We can extend the `FontOptions` with an integer template argument to serve as a set of flags, the ones set:

```c++
template<unsigned FlagsSet> class FOI {
protected:
    enum Flags { BOLD = 1, ITALIC = 2, UNDERLINED = 4, STRIKE_THROUGH = 8 };
    unsigned char flagsSet = 0;

    explicit constexpr FOI(unsigned integer) noexcept: flagsSet(integer) {}

    constexpr unsigned flagVal(bool v, unsigned flag) const noexcept {
        return v ? (flagsSet | flag) : (flagsSet & ~flag);
    }

    template<unsigned> friend class FOI;

public:
    FOI() = default;
    FOI(const FOI &) = default;

    constexpr FOI<FlagsSet | BOLD> bold(bool v) const noexcept {
        return FOI<FlagsSet | BOLD>(flagVal(v, BOLD));
    }

    constexpr FOI<FlagsSet | ITALIC> italic(bool v) const noexcept {
        return FOI<FlagsSet | ITALIC>(flagVal(v, ITALIC));
    }

    constexpr FOI<FlagsSet | UNDERLINED> underlined(bool v) const noexcept {
        return FOI<FlagsSet | UNDERLINED>(flagVal(v, UNDERLINED));
    }

    constexpr FOI<FlagsSet | STRIKE_THROUGH> strike_through(bool v) const noexcept {
        return FOI<FlagsSet | STRIKE_THROUGH>(flagVal(v, STRIKE_THROUGH));
    }

    constexpr bool bold() const noexcept { return flagsSet & BOLD; }
    constexpr bool italic() const noexcept { return flagsSet & ITALIC; }
    constexpr bool underlined() const noexcept { return flagsSet & UNDERLINED; }
    constexpr bool strikeThrough() const noexcept { return flagsSet & STRIKE_THROUGH; }
};
```

Observe the "setters" do not set, they employ the "tried and true" functional programming technique of avoiding mutability by making a modified copy of the execution environment, but most importantly, **the modified copy has a different type!**, a new type that indicates the flag specified.

Regardless of how the flags are specified, we can require them all by doing something like this:

```c++
struct Font {
    // ...
    Font(const std::string &, int, FOI<15>);
```

The overload requires a `FOI` with all four flags specified.

An example of use,

```c++
Font makeBoldItalic12Font(const std::string &name) {
    return  { name, 12, FOI<0>{}.bold(true).underlined(false).italic(true).strike_through(false) };
}
```

This technique, idiom, is more general than just guaranteeing complete configurations, thus, I will name it for what it is rather than this particular use.  The fundamental characteristic is that the "setters" change the return type, hence it contains "return type" in the name; the variation of the return type is a traversal of the power set of the set of flags required, but these traversals converge toward the complete set, furthermore, once you specify a flag, you won't un-specify it, and the technique can be used for several sets of properties to be specified, hence the *covariance*.  "Power set covariant return type" template pattern idiom.

Quoting at length [the article that introduced this idiom](https://thecppzoo.blogspot.com/2016/01/achieving-ultimate-date.html),

> Go ahead and try to build a Font from less than the result of initializing all the flags, the compiler will reject the code.

> The technique before lets you:

> To not lose any performance (if in doubt, construct a test case and see the assembler)
> 1. Full battery of compile-time reasoning about the values
> 2. To set the flags by name
> 3. Regardless of order
> 4. To have partial sets of flags and communicate them all over
> 5. The ability to enforce that all the flags are needed, simply require the signature with the number that represents all bits set.
> 6. Performance gains from keeping the set consolidated

> In summary, **at the same time greater clarity and performance**.

## Recap

We have gained clarity and performance. Next, we can adapt the macro to take care of the new boilerplate, generalize the technique to configuration properties of arbitrary types; which will happen in a successive article.

Also, we should comment on a sleight of hand we've made here: treating bit sets as an integer or vice versa as it is convenient to us. This goes against the principles of strict types present in the language; I think we should treat at length the subject of how strict types, which their advocates advocate because of potential optimizations related to propagation of constants and proving memory aliasing does not occur, in reality, from my point of view, impedes implementation-defined type punning extremely useful for clarity and performance. In my original articles I discovered the "power set..." idiom while talking about something else; that discussion will be continued.

