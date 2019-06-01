---
layout: post
title: "Serenity C++ patterns: The Badge"
---

This post describes the C++ `Badge` template used to enhance member function access control in [Serenity Operating System](https://github.com/SerenityOS/serenity).

---

C++ divides class member functions into three separate categories for access control. Let's review them quickly:

**Public members**: Accessible to everyone, these make up the public interface of the class.

**Protected members**: Accessible to the class itself and its derived classes.

**Private members**: Accessible only to the class itself.

An outsider class or function can also be declared as a **friend**. This gives that outsider VIP access to all the private parts of the class.

---

Sometimes you find yourself adding an interface to a class that's only meant for a specific outsider to use. Let's use an example from the `VFS` (virtual file system) class in Serenity:

```cpp
class VFS {
    ...
public:
    void register_device(Device&);
    void unregister_device(Device&);
};
```

These functions are called by all `Device` objects when they are constructed and destroyed respectively. They allow the `VFS` to keep track of the available device objects so they can be opened through files in the `/dev` directory.

Now, nobody except `Device` should ever be calling `VFS::register_device()` or `VFS::unregister_device()`, but since the functions are public members of `VFS`, anyone with a `Device&` can call them.

A common technique for preventing others from accessing these functions would be making them private and adding `Device` as a friend of `VFS`:

```cpp
class VFS {
    ...
private:
    friend class Device;
    void register_device(Device&);
    void unregister_device(Device&);
};
```

This prevents outsiders except `Device` from calling those functions, but it also means that `Device` now has full access to the private parts of `VFS` which is not great. Classes having full friend access to each other has a tendency to lead to overly comfortable access patterns.

Here's the solution I've used for this in Serenity:

```cpp
class VFS {
    ...
public:
    void register_device(Badge<Device>, Device&);
    void unregister_device(Badge<Device>, Device&);
};
```

The interfaces are public, but now you have to provide a `Badge<Device>` if you want to call them. A `Badge<Device>` is a simple, empty object that can *only* be constructed by `Device`. It works like this:

```cpp
template<typename T>
class Badge {
    friend T;
    Badge() {}
};
```

Basically, a `Badge<T>` is an empty class with a private constructor, and the only one who can call the constructor is their best friend, `T`.

In the `Device` constructor, the code to call `VFS::register_device()` looks like this:

```cpp
Device::Device()
{
    VFS::the().register_device({}, *this);
}
```

That little `{}` constructs an empty `Badge` and we're allowed to make the call.

*Final note: The name `Badge` refers to how the caller has to show their identification badge before being allowed into the function. :^)*

Until next time!
