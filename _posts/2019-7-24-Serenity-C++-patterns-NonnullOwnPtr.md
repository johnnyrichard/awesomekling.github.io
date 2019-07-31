---
layout: post
title: "Serenity C++ patterns: NonnullOwnPtr"
---

This post describes the C++ `NonnullOwnPtr` template used to enforce non-nullity of single-owner object pointers in the [Serenity Operating System](https://github.com/SerenityOS/serenity).

---

You're probably familiar with `std::unique_ptr`, the single-owner smart pointer in the C++ standard library.

In Serenity , the equivalent smart pointer is called `OwnPtr`, a name I've borrowed from the WebKit project (and although WebKit has long since switched to `std::unique_ptr`, I always felt that `OwnPtr` looked more beautiful.)

If you read my post on [using references instead of pointers](https://awesomekling.github.io/Serenity-C++-patterns-References-instead-of-Pointers/), you know I'm a fan of keeping `nullptr` out of sight (and mind) whenever possible, and **compile-time is the best time to catch bugs!**

Now, an `OwnPtr` can be null (just like `std::unique_ptr`) and that's perfectly reasonable. However, there are many situations where we know for sure that a pointer is not going to be null. For example:

* When we've just constructed a new object.
* When we're returning a pointer that we know isn't null.
* When a function argument is required to be null, and it's really up to the caller to make sure he's not passing us `nullptr`.

In all of these cases, we can use the handy `NonnullOwnPtr` smart pointer. Let me tell you about it...

#### NonnullOwnPtr as return type

Here's a simple example:

```cpp
NonnullOwnPtr<Object> create_object()
{
    if (something)
        return nullptr; // Would not compile.
    return make<Object>();
}

auto object = create_object();
if (!object) // Would not compile.
    return;
object->do_something();
```

Note that we're using `make<T>`, which is a helper that constructs a new object via `new T(...)` and returns it wrapped in a `NonnullOwnPtr`.

Because the `create_object()` function returns `NonnullOwnPtr`, it's not valid to return `nullptr`, and the compiler will prevent you from doing it.

It's also not valid to null-check a `NonnullOwnPtr` since it's never null.

Sadly it's not possible to override the dot(`.`) operator in C++ (yet), so we're forced to use `->` for dereferencing the `NonnullOwnPtr`. *(This bothers me more than I'm proud to admit, but whatcha gonna do...)*

#### NonnullOwnPtr as argument type

Let's do another example where `NonnullOwnPtr` is used for an argument:

```cpp
void Object::set_bar(NonnullOwnPtr<Bar>&& bar)
{
    m_bar = move(bar);
}

auto object = create_object();
object->set_bar(make<Bar>()); // Cool!
object->set_bar(nullptr); // Would not compile.
```

The `set_bar` function declaration now explicitly prevents callers from passing `nullptr` (or a regular `OwnPtr&&`, since those *may* be null.)

#### What about moved-from pointers?

Astute readers and paranoid veterans might now be asking "well, what about a moved-from `NonnullOwnPtr` though? If it can't be null, then what is it?"

The answer is that they are "invalid", but internally `nullptr`. I'm using [my Clang consumable annotation technique](https://awesomekling.github.io/Catching-use-after-move-bugs-with-Clang-consumed-annotations/) to have the compiler  generate a warning if a `NonnullOwnPtr` is used (other than invoking the destructor) after it's been moved from.

It's not perfect, but AFAIK it's the best we can do with the language support available today.

#### Bonus: NonnullOwnPtrVector

`NonnullOwnPtr` also has a complementary template called `NonnullOwnPtrVector`. It inherits from `Vector<NonnullOwnPtr<T>>` and enhances it by overloading all the accessors and making them return `T&` rather than `NonnullOwnPtr<T>&`.

This allows you to write very pleasant-looking code with dots(`.`) instead of arrows(`->`):

```cpp
void foo_them_all(const NonnullOwnPtrVector<Object>& objects)
{
    if (objects.is_empty())
        return;

    if (objects.first().is_fancy())
        dbg() << "The first object is fancy!";

    for (auto& object : objects)
        object.foo(); // Look, no ->!
}
```

Pretty neat, no? :^)

#### So where's the code?

* [serenity/AK/NonnullOwnPtr.h](https://github.com/SerenityOS/serenity/blob/master/AK/NonnullOwnPtr.h)
* [serenity/AK/NonnullOwnPtrVector.h](https://github.com/SerenityOS/serenity/blob/master/AK/NonnullOwnPtr.h)

Note: The `CONSUMABLE`, `SET_TYPESTATE`, `RETURN_TYPESTATE` and `CALLABLE_WHEN` macros can be found in [serenity/AK/Platform.h](https://github.com/SerenityOS/serenity/blob/master/AK/Platform.h)

Until next time!
