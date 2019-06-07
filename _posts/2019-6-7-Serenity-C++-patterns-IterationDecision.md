---
layout: post
title: "Serenity C++ patterns: IterationDecision"
---

This post describes the C++ `IterationDecision` pattern used to implement flow control during callback-based iteration in the [Serenity Operating System](https://github.com/SerenityOS/serenity).

---

When iterating over something in C++, your code might look something like this:

```cpp
for (auto& entry : m_entries) {
    if (entry.is_irrelevant())
        continue;
    if (entry.type() == Entry::Type::Cool) {
        if (do_stuff(entry))
            break;
    }
}
```

In the above example, we're using `continue` to skip over some entries, and `break` to stop iterating early instead of visiting any more entries.

Okay, so..

The Serenity codebase makes frequent use of custom `for_each_foo()` iteration helpers that take a callback parameter and invoke it for each iteration step.

Such a helper might be implemented like this:

```cpp
template<typename Callback>
void for_each_entry_of_type(Type type, Callback callback)
{
    for (auto& entry : m_entries) {
        if (entry.is_irrelevant())
            continue;
        if (entry.type() != type)
            continue;
        if (callback(entry) == IterationDecision::Break)
            break;
    }
}
```

Since the real "body" of the loop now happens in the `callback` function, we can't use the `continue` and `break` statements to control the loop in `for_each_entry`. The only way we can communicate with that loop is via a return value from the callback.

So that's exactly what we do! In `<AK/IterationDecision.h>` we have this enum:

```cpp
enum class IterationDecision { Continue, Break };
```

When you call one of those `for_each_foo()` functions, you just need to make the `callback` function inform the loop what to do next by using `return` statements instead of `continue` or `break`:

```cpp
for_each_entry_of_type(Entry::Type::Cool, [](auto& entry) {
    if (do_stuff_with(entry))
        return IterationDecision::Break;
    return IterationDecision::Continue;
}
```

This pattern can sometimes be a little more verbose than "native" loops, but it allows us to encapsulate iteration logic, which means that classes can expose public interfaces for iteration without having to show you the data structures inside.

Until next time!
