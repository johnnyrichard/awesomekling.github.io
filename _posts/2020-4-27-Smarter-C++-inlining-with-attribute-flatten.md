---
layout: post
title: "Smarter C/C++ inlining with __attribute__((flatten))"
---

This post describes a compile-time technique for getting the benefits of aggressive inlining in hot code while protecting cool code from its downsides.

---

Hello friends!

A common technique for improving performance of hot code in C/C++ is to inline the hottest functions called. While it often helps make things faster, there are some downsides to inlining. Let's quickly review the pros & cons:

**Pros of inlining:**

* Removes function call overhead (yay!)
* May reveal additional optimization opportunities (sometimes yay!)

**Cons of inlining:**

* Increases program size (boo!)
* May reduce cache locality (sometimes boo!)
* May increase build times (boo!)

When compiling with optimizations, the compiler usually makes pretty reasonable choices about which functions to inline. It uses a combination of heuristics, with function size being the most important one AFAIK.

## Manual inlining with `__attribute__((always_inline))`

However, sometimes *you* know some code is **hot** and the compiler has no idea. This is usually when `__attribute__((always_inline))` comes in. If you add this attribute to a function, that function will now be inlined wherever it is called, even when the compiler would normally have dismissed it as too large or otherwise unsuitable.

Here's a contrived example of a very common scenario in larger codebases:

```cpp
__attribute__((always_inline)) inline void do_thing(int input)
{
    // this code is always inlined at the call site
}

void hot_code()
{
    // the program spends >80% of its runtime in this function
    while (condition) {
        ...
        do_thing(y);
        ...
    }
}
```

The above is all well and good, but what happens when `do_thing()` is a popular function that gets called a lot?

```cpp
void cool_code()
{
    // the program spends <5% of its runtime in this function
    ...
    do_thing(a);
    do_thing(b);
    do_thing(c);
}
```

Now the `cool_code()` function gets three copies of `do_thing()` inlined into it, invoking all of the cons from the list we made above (larger program size, worse cache locality, longer build time.)

## Targeted flattening instead of global inlining

Now for the trick! Both GCC and Clang support `__attribute__((flatten))`. Putting it on a function causes all of its callees to be inlined into it. It's dead simple.

```cpp
void do_thing(int input)
{
    // this code is not always inlined at the call site
}

__attribute__((flatten)) void hot_code()
{
    // the program spends >80% of its runtime in this function
    while (condition) {
        call_something();   // inlined!
        do_thing(y);        // inlined!
        other_thing();      // also inlined!
    }
}

void cool_code()
{
    // the program spends <5% of its runtime in this function
    ...
    do_thing(a);            // not inlined!
    do_thing(b);            // not inlined!
    do_thing(c);            // guess!
}
```

*Note: Functions with `__attribute__((noinline))` will not be inlined. The same goes for functions where the compiler can't see the body.*

## In conclusion

`__attribute__((flatten))` lets you opt in to the pros of aggressive inlining on a per-function basis, while protecting the rest of your program from the cons!

Until next time! :^)
