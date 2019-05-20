---
layout: post
title: How Serenity system calls work
---

This post describes how system calls are implemented in the [Serenity Operating System](https://github.com/SerenityOS/serenity).

So let's say a userspace program wants to read from `stdin`, so it calls `read(STDIN_FILENO, buffer, sizeof(buffer))`. What happens next? Let's dive in!

The first place to look is the LibC implementation of `read()`, since that's the function being called:

    ssize_t read(int fd, void* buf, size_t count)
    {
        int rc = syscall(SC_read, fd, buf, count);
        __RETURN_WITH_ERRNO(rc, rc, -1);
    }

All the system calls in LibC are forwarded to the kernel using the `syscall()` helper. It's actually a macro that expands to `Syscall::invoke(...)`, which comes in four different versions, depending on how many arguments you want to pass.

The second line uses the `__RETURN_WITH_ERRNO()` helper whose purpose is to update `errno` based on the result from the kernel. Most system calls return an integer, with negative numbers indicating that an error occurred. Negative return values are inverted and stored into `errno`, and then the function returns `-1`. If there was no error, then the return value from the kernel is passed on to the caller, and `errno` is set to `0`.

Serenity is currently x86-only, and kernel system calls are done by calling `int 0x82` with the following inputs:

* `eax`: System call index
* `edx`: Argument 1
* `ecx`: Argument 2
* `ebx`: Argument 3

There is only one output:

* `eax`: Return value from kernel

So the invocation of `syscall(SC_read, ...)` above turns into the following assembly code:

    b8 05 00 00 00          mov    eax,0x5
    8b 55 08                mov    edx,DWORD PTR [ebp+0x8]
    8b 4d 0c                mov    ecx,DWORD PTR [ebp+0xc]
    8b 5d 10                mov    ebx,DWORD PTR [ebp+0x10]
    cd 82                   int    0x82

Note that `SC_read` takes three inputs, so all three argument registers are used. When a system call takes less than three inputs, the other argument registers are ignored.

When the `int 0x82` instruction kicks in, we are teleported from userspace code into the kernel, specifically into the `syscall_trap_handler` function:

    asm(
        ".globl syscall_trap_handler \n"
        "syscall_trap_handler:\n"
        "    pusha\n"
        "    pushw %ds\n"
        "    pushw %es\n"
        "    pushw %fs\n"
        "    pushw %gs\n"
        "    pushw %ss\n"
        "    pushw %ss\n"
        "    pushw %ss\n"
        "    pushw %ss\n"
        "    pushw %ss\n"
        "    popw %ds\n"
        "    popw %es\n"
        "    popw %fs\n"
        "    popw %gs\n"
        "    mov %esp, %eax\n"
        "    call syscall_trap_entry\n"
        "    popw %gs\n"
        "    popw %gs\n"
        "    popw %fs\n"
        "    popw %es\n"
        "    popw %ds\n"
        "    popa\n"
        "    iret\n"
    );

It may look chunky, but all that's really happening here is that the CPU general-purpose and segment registers are pushed onto the stack, and the data segment registers are loaded with kernel data selectors. Once that is set up, we call `syscall_trap_entry()` with a pointer to the register dump on the stack.

So next we end up here:

    void syscall_trap_entry(RegisterDump& regs)
    {
        current->process().big_lock().lock();
        dword function = regs.eax;
        dword arg1 = regs.edx;
        dword arg2 = regs.ecx;
        dword arg3 = regs.ebx;
        regs.eax = Syscall::handle(regs, function, arg1, arg2, arg3);
        if (auto* tracer = current->process().tracer())
            tracer->did_syscall(function, arg1, arg2, arg3, regs.eax);
        current->process().big_lock().unlock();
    }

`current` is a kernel global pointer to the currently executing `Thread`. `current->process()` is the currently executing `Process`.

The first thing you'll notice is that this function acquires the current process's "big lock" on entry, and releases it on exit. The big lock prevents other threads in the same process from entering the kernel at the same time. *(Note: The lock may be temporarily released before returning from this system call, in case the calling thread blocks inside the kernel.)*

Next up, we put the relevant register values into clearly named variables to make it clear what they are, and finally call `Syscall::handle()`. The return value is then stored back into `eax` in the register dump on the stack, since the CPU state will be restored from this dump when we return to userspace.

If the current process has a system call tracer attached to it, we inform it of the call that just happened, and what the return value was. *(This is currently how `strace` is implemented in Serenity.)*

Once we return from this function, we'll go back to the assembly kludge above, where `iret` will teleport us back to userspace, and the journey is over.

We're not quite done yet though, so let's keep going. In `Syscall::handle()`, all that really happens is a big `switch` on the function index. In our case, that's `SC_read` so we end up here:

    static dword handle(RegisterDump& regs, dword function, dword arg1, dword arg2, dword arg3)
    {
        ...
        switch (function) {
            ...
            case Syscall::SC_read:
                return current->process().sys$read(
                    (int)arg1,
                    (byte*)arg2,
                    (ssize_t)arg3
                );
        }
    }

The code above will cast the three input arguments to the types expected by `Process::sys$read` and then transfer control there.

And finally, we end up in some "business logic":

    ssize_t Process::sys$read(int fd, byte* buffer, ssize_t size)
    {       
        if (size < 0)
            return -EINVAL;
        if (size == 0)
            return 0;
        if (!validate_write(buffer, size))
            return -EFAULT;
        auto* descriptor = file_descriptor(fd);
        if (!descriptor)
            return -EBADF;
        if (descriptor->is_blocking()) {
            if (!descriptor->can_read()) {
                current->block(Thread::State::BlockedRead, *descriptor);
                if (current->m_was_interrupted_while_blocked)
                    return -EINTR;
            }
        }   
        return descriptor->read(buffer, size);
    }       

Explaining how reading from a file descriptor works is beyond the scope of this post, so we've now arrived at our destination.

I hope you found this understandable and/or interesting. I encourage you to look deeper into the sources if you want to know more. They are always up-to-date in the GitHub repository. :)

Until next time!

