---
layout: post
title: "Serenity C++ patterns: NetworkOrdered"
---

This post describes the C++ `NetworkOrdered<T>` template used to simplify working with network packet structures in the [Serenity Operating System](https://github.com/SerenityOS/serenity).

---

If you're familiar with C network programming, you already know the `htonl()` & `ntohl()` function family.

They are used for converting values between "network order" and "host order", where "network order" really means "big endian". So on a big endian machine, these functions are no-ops, but on little endian machines (such as the x86 targeted by Serenity), they perform a byte-order swap.

It's called "network order" because the packet formats of protocols like **IPv4** and **TCP** store numeric values in big-endian format. It's also used in file formats such as the **PNG** image format.

Forgetting to call `ntohl()` in a single place can easily lead to hard-to-diagnose problems. It also doesn't help that the value 0 doesn't change when byte-swapped, so you might not even be seeing the effect of that missing byte-swap yet.

For Serenity, I've fashioned a simple C++ template to help avoid problems like this. It's called `NetworkOrdered<T>` and here's how you can use it:

```cpp
NetworkOrdered<dword> foo;
foo = 0x12345678; // foo stores 0x78563412 internally
dword bar = foo; // bar is now 0x12345678
dword baz = foo - 1; // baz is now 0x12345677
```

This is incredibly useful when defining things like the **TCP** packet header, which has the following members:

```cpp
class [[gnu::packed]] TCPPacket {
    ...
    NetworkOrdered<word> m_source_port;
    NetworkOrdered<word> m_destination_port;
    NetworkOrdered<dword> m_sequence_number;
    NetworkOrdered<dword> m_ack_number;
    NetworkOrdered<word> m_flags_and_data_offset;
    NetworkOrdered<word> m_window_size;
    NetworkOrdered<word> m_checksum;
    NetworkOrdered<word> m_urgent;
}
```

The Serenity kernel works with raw network buffers a lot, and so this allows us to cast some buffer pointer to a `TCPPacket*` and use the variables in computations without worrying about the byte order.

Here's how the template works:

```cpp
template<typename T>
inline T swap_bytes_if_needed(T value)
{
#if BYTE_ORDER == LITTLE_ENDIAN
    if constexpr (sizeof(T) == 8)
        return __builtin_bswap64(value);
    if constexpr (sizeof(T) == 4)
        return __builtin_bswap32(value);
    if constexpr (sizeof(T) == 2)
        return __builtin_bswap16(value);
    if constexpr (sizeof(T) == 1)
        return value;
#else
    return value;
#endif
}

template<typename T>
class [[gnu::packed]] NetworkOrdered {
public:
    NetworkOrdered() {}
    NetworkOrdered(const T& host_value)
        : m_network_value(swap_bytes_if_needed(host_value))
    {
    }

    operator T() const
    {
        return swap_bytes_if_needed(m_network_value);
    }

private:
    T m_network_value { 0 };
};
```

Dead simple, right?

Note that the `[[gnu::packed]]` is required to prevent warnings about embedding "unpacked non-POD fields" in other packed structs.

Until next time!
