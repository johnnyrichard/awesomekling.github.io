---
layout: page
title: Frequently Asked Questions
permalink: /faq/
---

These are some very frequently asked questions about me ([Andreas Kling]({{ site.url }})) and the [SerenityOS](https://serenityos.org/) project.

## What is your job?

As of May 2021, [I'm working on SerenityOS full time](https://awesomekling.github.io/I-quit-my-job-to-focus-on-SerenityOS-full-time/).

In the past, I have worked as a programmer at many small and large tech companies, including Apple (2011-2017) and Nokia (2009-2011).

## What Linux distro are you using?

[Ubuntu MATE](https://ubuntu-mate.org/).

## What IDE are you using?

I'm using the [CLion C++ IDE from JetBrains](https://jetbrains.com/clion/).

## What keyboard are you using?

It's a [Logitech K740](https://www.logitech.com/en-us/product/illuminated-keyboard-k740). I bought it primarily for the pleasantly low typing noise, but it's also quite nice to type on.

## Why are you building SerenityOS? / What is your goal with SerenityOS?

I started building SerenityOS to keep myself busy after finishing a 3-month drug addiction rehab program in 2018.

My goal is to build a desktop system for myself to use. I'm not trying to attract users or appeal to anyone other than myself.

Since I started, other people have joined the project with their own motivations, and I have no control over what motivates them.

## Will you implement [thing] for SerenityOS?

I don't have a plan for features/technologies/etc to implement.

## Will it be possible to do [thing] in SerenityOS?

If someone implements it, yes. If not, no. If you want something to happen, it's up to you to make it happen.

## Does SerenityOS have a package manager?

No, SerenityOS does not have a package manager. The project uses a monorepo approach, meaning that all software is built in the same style and using the same tools. There is no reason to have something like a package manager because of this.

However there are ports which can be found in the Ports directory. A port is a piece of software that can optionally be installed, might have not been built by us but supports running on SerenityOS. They act quite similar to packages, coming with an install script each.

Currently when running the system in a virtual machine, ports need to be cross compiled on the host and added to the file system image before booting. Then its also possible to configure the build system to in- or exclude components from a build.

## Why do you use QEMU for development?

Because it's the most convenient way to make rapid progress at this stage. The fast edit-compile-boot-and-test cycle is extremely fast.

## Does SerenityOS run on bare metal?

Yes. Check out the [SerenityOS installation guide](https://github.com/SerenityOS/serenity/blob/master/Documentation/BareMetalInstallation.md).

## When will SerenityOS be self-hosting?

It depends on what you mean by self-hosting. We've already built the system on itself many times in the past, it's just not a workflow that is actively maintained since everyone is focused on other stuff.

## Why is the system 32-bit?

There is a 64-bit port (x86\_64) as well.

## What do you think about Rust / Zig / other programming language?

Unless I have written over 100'000 lines of code in a programming language, my opinion is not worth hearing.

Accordingly, I have no opinion worth hearing about Rust or Zig.

## How do I learn to build an OS?

Check out the [OSDev Wiki](https://wiki.osdev.org/Main_Page).

## Will you make tutorials about OS development?

No, I'm not making educational content for beginners. There are plenty of resources for beginners already, and I'm focusing on content for more advanced programmers.
