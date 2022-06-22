---
title: Revisiting Stateful Metaprogramming in C++20
---

**Disclaimer**: Stateful metaprogramming is well-known as one of those things that almost certainly shouldn't be used in production code. Even in the best cases, its correctness is debatable. As such, this article is solely for purposes of personal curiosity.

## Introduction

Recently I discovered the rather arcane C++ trick known as "stateful metaprogramming", in which one manipulates friend functions and template instantiation to introduce utilisable state into compilation. The result: some surprising but very interesting behaviour, such as impure `constexpr` functions:

```c++
constexpr bool func() { /* something */ }

constexpr bool a = func();
constexpr bool b = func();
// Think this assertion always fails? With stateful metaprogramming, think again.
static_assert(a != b);
```

The trick is not new; it has been examined and discussed by several others since at least the C++14 days:

- [*Non-constant constant-expressions in C++*](https://b.atch.se/posts/non-constant-constant-expressions/) (2015), by Filip Roséen
- [*How to implement a constant-expression counter in C++*](https://b.atch.se/posts/constexpr-counter/) (2015), by Filip Roséen
- [*How to implement a compile-time meta-container in C++*](https://b.atch.se/posts/constexpr-meta-container/) (2015), by Filip Roséen
- [*The C++ Type Loophole*](https://alexpolt.github.io/type-loophole.html) (2017), by Alexandr Poltavsky
- [CWG issue 2118](https://www.open-std.org/jtc1/sc22/wg21/docs/cwg_active.html#2118) (2015)
- [*Is stateful metaprogramming ill-formed (yet)?*](https://stackoverflow.com/questions/44267673/is-stateful-metaprogramming-ill-formed-yet) (2017), on Stack Overflow

I would highly recommend reading the above articles first - they are quite enlightening if you, like myself a few days ago, never considered the possibility of storing state within the compilation process. This article will assume you have a basic understand of how stateful metaprogramming works.

Aside from the obvious mindblowing qualities of the technique, two aspects stood out to me from the articles I came across:

- The compiler support was somewhat finicky and dodgy, being inconsistent across different compilers or different versions. Many of the code examples I found do not work on the latest versions of the major compilers.
- It was quite easy to accidentally trigger undefined behaviour or (my personal dread) ill-formed no diagnostic required.

As someone who has been deep in C++20 for the last several months, I thought: the C++ language and compilers have progressed significantly since C++14 - can we exploit these developments to improve upon our existing usages of stateful metaprogramming? Or, at very least, can we learn anything interesting or valuable while re-examining this technique under the eye of C++20?  
There have been unsatisfyingly few such discussions, or at least few that are comprehensive. This article is my attempt.

Before we begin, please note that this article comes from the perspective of someone who is not an expert on the C++ Standard. What I've written is accurate to the best of my understanding, but I don't have the skill to justify every single detail with quotes from the Standard. If you are someone who does possess such skill, please feel free to correct me.

## Part 1: Nonconstant Constant Expressions

Let's start by revisiting the minimal example of stateful metaprogramming: a "nonconstant" constant expression that will have a different value on the first evaluation to subsequent evaluations, without any user-provided state. Such a task requires one bit of information as state.  
In this example I will use the evaluation of a `constexpr` variable, but one could alternatively create a nullary `constexpr` function that achieves the same.

### Code

To cut straight to the chase, here is the C++20 code I came up (explanation will follow):

```c++
auto flag(int);     // E1


template<bool B> requires (!B)
struct setter {
    friend auto flag(int) {}        // E2

    static constexpr bool b = B;
};


// E3
template<bool FlagVal>
[[nodiscard]]
consteval auto nonconstant_constant_impl() {
    if constexpr (FlagVal) {
        return true;
    }
    else {
        // E3.1
        setter<FlagVal> s;
        return s.b;
    }
}


// E4
template<
    auto Arg = 0,
    bool FlagVal = requires { flag(Arg); },             // E4.1
    auto Val = nonconstant_constant_impl<FlagVal>()     // E4.2
>
constexpr auto nonconstant_constant = Val;
```

Live demo: [https://godbolt.org/z/a31n5xqMc](https://godbolt.org/z/a31n5xqMc)

### Explanation

The line marked `E1` declares a function `flag()` which will effectively store one bit of state. The value of the bit is equal to whether or not a call to `flag()` is well-formed. `flag()` has deduced return type, but at this point no definition to deduce a type, so a call is currently ill-formed.

The code marked `E2` provides a definition for `flag()` via a friend function in a class template `setter`. Once `setter` is instantiated for the first time, the definition will be visible in the enclosing namespace. As a result, calls to `flag()` will become well-formed - effectively changing the value of our bit of state.

The code marked `E3` is a helper function `nonconstant_constant_impl()` for enacting our nonconstant constant expression. Its template argument `FlagVal` will be the value of the one bit of state (initially `false`, then `true` after the first evaluation - you will see how shortly). The return value is just the value of `FlagVal`, but with a side effect when `FlagVal` is `false`. In that case, the code marked `E3.1` is instantiated, instantiating `setter` and changing the value of the bit of state.  
Here `FlagVal` is used as the template argument for `setter`, preventing the compiler from eagerly instantiating `setter`. And the use of `setter::b` prevents the compiler from deferring the instantiation of `setter`.

The code marked `E4` is the `constexpr` variable `nonconstant_constant` that achieves our end goal. On the line marked `E4.1` we compute the value of our state bit by checking if a call to `flag()` is well-formed with a `requires` expression. Using unqualified lookup and passing a template argument `Arg` forces the compiler to delay the check to the point of instantiation (in particular, due to ADL). We then invoke `nonconstant_constant_impl()` on line `E4.2` to obtain the value assigned to `nonconstant_constant` and possibly change the value of the state bit, as explained above.  
The entire contraption is performed within the template arguments of `nonconstant_constant`, meaning that any usage of the variable will force it to be evaluated, consistently.  

In summary:

1. The validity of a call to `flag()` stores one bit of state: ill-formed → 0; well-formed → 1.
2. The validity of a call to `flag()` depends on whether or not `setter` has been instantiated.
3. On the first evaluation of `nonconstant_constant`, `setter` has not been instantiated and the call to `flag()` is ill-formed.
4. The value of `nonconstant_constant` is `false`.
5. `setter` is instantiated.
6. On the second evaluation of `nonconstant_constant`, the call to `flag()` is well-formed.
7. The value of `nonconstant_constant` is `true`.

The end result:

```c++
constexpr bool a = nonconstant_constant<>;      // First evaluation in this translation unit
constexpr bool b = nonconstant_constant<>;
// This assertion passes.
static_assert(a != b);
```

### Discussion

A necessary acknowledgement: you may consider this code to be cheating, since technically there is not a single `nonconstant_constant` which evaluates to different values, but multiple template instantiations which each have a different a value. A fair point, although otherwise it is truly impossible to create a nonconstant constant expression as far as I'm aware.

I mentioned that `nonconstant_constant` could be a `constexpr` function as well, yet I have chosen a `constexpr` variable. The reason is that certain uses of a function template (such as taking the address) on certain compilers seem to elide instantiation of the body. (Someone more knowledgeable can say whether or not this is Standard-conforming behaviour, but nevertheless it does occur in practice.) I found more consistent behaviour with `constexpr` variables.

What I find most remarkable about this code is that it seems to be valid. While I'm no language lawyer, the fact that it works on all three major compilers (GCC, Clang, MSVC) is compelling.

Additionally, I'd argue the code is quite readable and "safe" - at least as far as metaprogramming goes - partially thanks to new C++20 features.  
In the previous examples of stateful metaprogramming that I saw, the defined-ness of the flag function was checked with SFINAE or the `noexcept` operator. Now in C++20, we can do it more elegantly and reliably with a `requires` expression.  
There are also extra safety mechanisms that are easily implemented to prevent misuse and unexpected behaviour. First, `setter` can only be instantiated with one template argument, `B=false`, due to the associated `requires` clause. This setup prevents multiple instantiations of `setter`, which would yield redefitions of `flag()` within a translation unit. Second, the helper function `nonconstant_constant_impl()` is declared `consteval` and `nodiscard`, dissuading the previously mentioned usage scenarios that can inconsistently elide instantiation of the function body.

## Part 2: Compile-time Counting

Now let's move on to a more useful and complex example: a compile-time counter. That is, a `constexpr` variable which will evaluate to sequential increasing integers on each usage, without any additional state provided by the user (again, one could also create a nullary `constexpr` function). This task requires storing multiple bits of information.  

### Code

Here is the C++20 code I came up with:

```c++
template<unsigned N>
struct reader {
    friend auto counted_flag(reader<N>);        // E1
};


template<unsigned N>
struct setter {
    friend auto counted_flag(reader<N>) {}      // E2

    static constexpr unsigned n = N;
};


// E3
template<
    auto Tag,               // E3.1
    unsigned NextVal = 0
>
[[nodiscard]]
consteval auto counter_impl() {
    constexpr bool counted_past_value = requires(reader<NextVal> r) {
        counted_flag(r);
    };

    if constexpr (counted_past_value) {
        return counter_impl<Tag, NextVal + 1>();            // E3.2
    }
    else {
        // E3.3
        setter<NextVal> s;
        return s.n;
    }
}


template<
    auto Tag = []{},                // E4
    auto Val = counter_impl<Tag>()
>
constexpr auto counter = Val;
```

Live demo: [https://godbolt.org/z/M4bWsrcT5](https://godbolt.org/z/M4bWsrcT5)

### Explanation

The principles at play here are largely the same as the previous example, except that this time we store more than one bit of state. Each bit is associated with a value of the counter, indicating whether or not that particular value has been counted yet. The reason for this scheme is that once a bit has been set, it cannot be changed because functions cannot be made undefined once defined.

The code marked `E1` declares a family of overloaded functions `counted_flag()` via the class template `reader<N>`; one overload for each counter value `N`. Using `reader<N>` as the function's argument type enables lookup outside the class through ADL.

The code marked `E2` provides the definitions for `counted_flag()`. Instantiating `setter<N>` indicates counter value `N` has been counted.

At `E3` the function `counter_impl()` implements the counter logic. On each call, it returns the next value of the counter. Our bits of state cannot directly provide the counter value, but they do indicate which values have already been counted, so we iterate from 0 until the first uncounted value - the counter's next value - is found. The iteration is done here with recursion on line `E3.2`. On the final recursive call, at the code marked `E3.3`, the value is flagged as counted by instantiating `setter` with `N` equal to the counter value.

However, there is a hitch with `counter_impl()`: after initial instantiation with a particular value of `NextVal`, the compiler caches the instantiation and return value (i.e. counter value) is permanently fixed. This is where the extra template parameter `Tag` on the line marked `E3.1` comes into play. Defaulting `Tag` to be a lambda on line `E4` creates a new lambda with a new type upon every evaluation, necessitating reinstantiation of the function body.

The end result:

```c++
// These assertions pass.
static_assert(counter() == 0);      // First evaluation in this translation unit
static_assert(counter() == 1);
static_assert(counter() == 2);
static_assert(counter() == 3);
// And so on...
```

### Discussion

Again, this code works on all three major compilers. That said, I am less confident that its behaviour is mandated by the Standard, as I have seen claims that the order of instantiation within a translation unit is not specified (i.e. the counter may not be strictly required to count in the order we expect).  
Also, GCC produces this warning on the declaration of `counted_flag()`:

```
warning: friend declaration 'auto counted_flag(reader<N>)' declares a non-template function [-Wnon-template-friend]
```

which I presume is related to the potential traps of stateful-metaprogramming-like behaviour. But as far as I can tell, the code here does not fall into any of those traps, or it uses them to our advantage.  
I will leave it to the language experts to weigh in on these points more scientifically.

With regards to the unique lambda trick, I consider it to be an inelegant hack (yes, as if everything else in this article is not a hack...), although I could not figure out an alternative implementation. I would be very interested to see such an alternative, if it exists.

## Part 3: Compile-time List

For the final example, we will examine a compile time type list that supports appending, without any user-provided state. This code is quite a bit more complex than the previous two examples, but the fundamental concepts are similar.

### Code

The C++20 code I came up with:

```c++
#include <concepts>
#include <type_traits>


// E1
template<typename...>
struct type_list {};


// E2
template<class TypeList, typename T>
struct type_list_append;

template<typename... Ts, typename T>
struct type_list_append<type_list<Ts...>, T> {
    using type = type_list<Ts..., T>;
};


// E3
template<unsigned N, typename List>
struct state_t {
    static constexpr unsigned n = N;
    using list = List;
};


namespace {
    struct tu_tag {};           // E4
}


template<
    unsigned N,
    std::same_as<tu_tag> TUTag
>
struct reader {
    friend auto state_func(reader<N, TUTag>);
};


template<
    unsigned N,
    typename List,
    std::same_as<tu_tag> TUTag
>
struct setter {
    // E5
    friend auto state_func(reader<N, TUTag>) {
        return List{};
    }

    static constexpr state_t<N, List> state{};
};


template struct setter<0, type_list<>, tu_tag>;     // E6


// E7
template<
    std::same_as<tu_tag> TUTag,
    auto EvalTag,
    unsigned N = 0
>
[[nodiscard]]
consteval auto get_state() {
    constexpr bool counted_past_n = requires(reader<N, TUTag> r) {
        state_func(r);
    };

    if constexpr (counted_past_n) {
        return get_state<TUTag, EvalTag, N + 1>();
    }
    else {
        // E7.1
        constexpr reader<N - 1, TUTag> r;
        return state_t<N - 1, decltype(state_func(r))>{};
    }
}


// E8
template<
    std::same_as<tu_tag> TUTag = tu_tag,
    auto EvalTag = []{},
    auto State = get_state<TUTag, EvalTag>()
>
using get_list = typename std::remove_cvref_t<decltype(State)>::list;


// E9
template<
    typename T,
    std::same_as<tu_tag> TUTag,
    auto EvalTag
>
[[nodiscard]]
consteval auto append_impl() {
    using cur_state = decltype(get_state<TUTag, EvalTag>());            // E9.1
    using cur_list = typename cur_state::list;
    using new_list = typename type_list_append<cur_list, T>::type;      // E9.2
    setter<cur_state::n + 1, new_list, TUTag> s;                        // E9.3
    return s.state;                                                     // E9.4
}


// E10
template<
    typename T,
    std::same_as<tu_tag> TUTag = tu_tag,
    auto EvalTag = []{},
    auto State = append_impl<T, TUTag, EvalTag>()
>
constexpr auto append = [] { return State; };           // E10.1
```

Live demo: [https://godbolt.org/z/rc6551WoE](https://godbolt.org/z/rc6551WoE)

### Explanation

The idea behind the mechanism is to associate a list of types with counter values. The list starts empty at counter value 0. When we perform the append operation, we add a new type to the list and associate the new list with the incremented counter value.

The first few lines of code are mostly uninteresting helper utilities.

`type_list` declared at the code marked `E1` just stores a parameter pack of types.  
`type_list_append` at the code marked `E2` is a type metafunction accepting a `type_list` and a type `T`, and returns a new `type_list` with `T` appended to the end.

`state_t` declared at the code marked `E3` represents the state of our list. It consists of a counter value and the associated type list.

The aspect of primary interest comes with the function `state_func()` at the code marked `E5`. This time, instead of a flag function which we only care about being defined or undefined, we also store information with the return type deduced in the definition. In this manner, a counter value `N` can be associated with the type `List`, which will be a `type_list` representing the list's current value.

The issue with this method is that now the body of `state_func()` is not entirely determined by the template arguments of `setter`, but instead depends on the compilation state of the current translation unit. Such a scenario can very easily violate ODR. The solution is to instantiate `state_func()` with a different template argument in each translation unit, yielding different specialisations. This is done with the `tu_tag` type declared at line `E4` within an unnamed namespace, and the `TUTag` template parameters.

The line marked `E6` provides the initial empty state of the list.

The code at `E7` implements a function to get the stored state as a `state_t`. It operates similar to the previous counter code, iterating until the current counter value is found. The only difference is that it then checks the return type of `state_func()` to obtain the stored type list (the code marked `E7.1`). We only know what counter value `N` we are up to once we have checked value `N+1`, hence the `N-1` at the code `E7.1`. (Note that `N` can never be 0 when computing `N-1`, because of line `E6`.)

The code marked `E8` is a utility for getting the current list from the stored state.

The function declared at `E9` implements the list append operation, which consists of obtaining the current state (`E9.1`), appending the specified type `T` to the list (`E9.2`), and the associating the next counter value with the new list (`E9.3`). The function returns the new list state (`E9.4`) to force instantiation of `setter` (otherwise, the return value is unused).

At the code marked `E10` is the user-facing wrapper for the append operation. This is a `constexpr` variable for reasons discussed previously, but evaluates to a lambda so it can be used nicely as a call expression (e.g. `append<int>();`) rather than a discarded id-expression (e.g. `append<int>;`) which may cause a warning.

The end result:

```c++
// All these assertions pass.

static_assert(std::same_as<get_list<>, type_list<>>);       // First usage in this translation unit

append<int>();
static_assert(std::same_as<get_list<>, type_list<int>>);

append<float>();
static_assert(std::same_as<get_list<>, type_list<int, float>>);
```

### Discussion

Sadly this usage of stateful metaprogramming is inherently less safe than the previous two examples due to the multiple return types of `state_func()`, and C++20 can't solve that. What C++20 can do, though, is help ensure correct usage of the `tu_tag` workaround. All the `TUTag` template parameters are constrained with `std::same_as` to only allow `tu_tag`, preventing accidentally passing another type and violating ODR. To be honest I have a slight concern that such a constraint is ill-formed NDR because its meaning changes in each translation unit (there is a surprising amount of ill-formed NDR introduced with concepts and constraints - go take a look). Hopefully it is well-formed because the constraint only takes effect upon instantiation.

It should be evident from this final example that stateful metaprogramming is quite powerful. After a list, only the sky is the limit.

## Conclusion

Well, that may have been the most exotic C++ code I have ever written. Yet I think there were definitely things to be learnt.

It seems as though it is possible to use stateful metaprogramming in C++20 in a more compliant manner. Or at least a manner which yields the behaviour we want consistently across the major compilers. Additionally, we can utilise new language features such as concepts and constraints to aid readability and assist in ensuring correctness and consistency.

Does this mean stateful metaprogramming is now safe? Is it perhaps more than just a toy construct? I believe the answer is still "no". As I have touched on a few times, I don't have 100% confidence that the code shown here is Standard-conforming. Certainly, the ISO C++ Working Group would like to disallow it in the future.  

If you have further thoughts on how the code here could be improved, how it is conforming, or how it is nonconforming, please let me know! This entire process of learning about stateful metaprogramming has been incredibly interesting and educational for me, and I would love to know more.

## Further Reading/Reference

Here are a few other articles which assisted my understanding of stateful metaprogramming and may be of interest:

- [Type loophole struct reader](https://github.com/alexpolt/luple/blob/master/type-loophole.h) (2017), by Alexandr Poltavsky
- [*How to Hack C++ with Templates and Friends*](https://ledas.com/post/857-how-to-hack-c-with-templates-and-friends/) (2020), by Andrey Karpovskii
- [*A foliage of folly*](https://dfrib.github.io/a-foliage-of-folly/) (2020), by David Friberg
