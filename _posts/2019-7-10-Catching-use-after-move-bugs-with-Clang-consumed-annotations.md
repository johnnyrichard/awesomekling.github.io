---
layout: post
title: "Catching use-after-move bugs with Clang's consumed annotations"
---

This post describes a compile-time technique for catching use-after-move bugs in modern C++. It's currently used to prevent some mistakes in the [Serenity Operating System](https://github.com/SerenityOS/serenity).

---

We were all excited when C++11 added move semantics. Finally we could work *together* with the compiler to avoid copies (instead of *against* the compiler, which is how it sometimes felt before.)

There is one footgun lurking with move semantics though. Once you `std::move` an object and pass it to someone who takes an rvalue-reference, your local object may very well end up in an invalid state. It'll still be destructible, of course, but it's no longer logically valid.

An obvious example is `std::unique_ptr`:

```cpp
auto object = std::make_unique<Object>();
auto other_object = std::move(object);
// Now the Object has moved into "other_object"
// and "object" is null.
other_object->do_something(); // This is fine.
object->do_something(); // Crash: nullptr dereference!
```

This kind of problem can be hard to spot in longer functions, and you find yourself wishing the compiler would give you a hand..
 
So I came up with a way to catch some of these bugs. It only works with the Clang compiler (AFAIK) and uses some obscure attributes listed under ["Consumed Annotation Checking"](https://clang.llvm.org/docs/AttributeReference.html#consumed-annotation-checking) in the Clang documentation.

The basic idea is: you mark a class as **consumable**. Objects of that class then exist one of three states: ***unconsumed***, ***consumed*** or ***unknown***. Using function attributes, you can read and modify this state on a per-object basis. And most importantly (for our needs), the compiler can generate a warning when a member function is called on an object in an unwanted state!

Let's write a little program to illustrate how this mechanism can be harnessed. First, an error-prone version:

```cpp
class Object {
public:
    Object() {}
    Object(Object&& other) { other.invalidate(); }
    void do_something() { assert(m_valid); }

private:
    void invalidate() { m_valid = false; }
    bool m_valid { true };
};

int main(int, char**)
{
    Object object;
    auto other = std::move(object);
    object.do_something();
    return 0;
}
```

The above code will assert in `do_something()`. There's nothing technically wrong with the program, so the compiler won't get in our way.

Now let's annotate the class with these attributes:

```cpp
class [[clang::consumable(unconsumed)]] CleverObject {
public:
    CleverObject() {}
    CleverObject(CleverObject&& other) { other.invalidate(); }

    [[clang::callable_when(unconsumed)]]
    void do_something() { assert(m_valid); }

private:
    [[clang::set_typestate(consumed)]]
    void invalidate() { m_valid = false; }

    bool m_valid { true };
};

int main(int, char**)
{
    CleverObject object;
    auto other = std::move(object);
    object.do_something();
    return 0;
}
```

We've made these three annotations:

* The `CleverObject` class is made **consumable**, with each object starting out in the ***unconsumed*** state.
* The `do_something()` functionfunction  must only be called on objects in the ***unconsumed*** state.
* The `invalidate()` function sets the state of the callee object to ***consumed***. Note that the `CleverObject(CleverObject&&)` move constructor calls `invalidate()` on the moved-from object, causing it to become ***consumed***.


Compiling the above code with `clang -Wconsumed` gives you this output:

```
clang -Wconsumed    test.cpp   -o test
test.cpp:36:12: warning: invalid invocation of method 'do_something' on object 'object'
      while it is in the 'consumed' state [-Wconsumed]
    object.do_something();
           ^
1 warning generated.
```

Pretty neat, huh? :^)

For a bigger example, you can see how I've implemented the [NonnullRefPtr](https://github.com/SerenityOS/serenity/blob/master/AK/NonnullRefPtr.h) smart pointer in Serenity. `NonnullRefPtr` is a variant of `RefPtr` (a reference counting smart pointer) that cannot be null. However, you are allowed to move from it, which puts it into an invalid (and ***consumed***) state.

Curiously, I couldn't find a single instance of anyone using these attributes for anything on the web. If you've seen them anywhere, I'd love to hear about it.

Until next time!
