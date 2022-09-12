---
layout: post
title: "Ladybird: A new cross-platform browser project"
---

This post describes the Ladybird browser, based on the LibWeb and LibJS engines from SerenityOS.

---

Since starting the [SerenityOS](https://serenityos.org) project in 2018, my goal has been "to build a complete desktop operating system to eventually use as my daily driver".

What started as 
[a little therapy project](https://awesomekling.github.io/I-quit-my-job-to-focus-on-SerenityOS-full-time/) for myself has blossomed into a huge OSS community with hundreds of people working on it all over the world. We've gone from *nothing* to a capable system with its own browser stack in the last 4 years.

Throughout this incredible expansion, my own goals have remained the same. Today I'm updating them a little bit: *in addition to building a new OS for myself, I'm also going to build a cross-platform web browser.*

## A browser is born

![This post in Ladybird]({{ site.url }}/assets/ladybird-thispost.png)

The [Ladybird](https://github.com/SerenityOS/ladybird) browser came to life on July 4th, when I recorded [a video of myself making a simple Qt GUI for the LibWeb browser engine](https://www.youtube.com/watch?v=X38MTKHt3_I). Thanks to some recent work by [Dex](https://github.com/dexesttp) and others, we had LibWeb building on Linux in headless mode, so I decided to push ahead and build a simple GUI around it.

I originally imagined Ladybird as a debugging tool that made it easier for people to remain in Linux while working on LibWeb if they wanted to. It's now two months later, and I find myself using Ladybird for most of my own browser development work.

At this point, we might as well tweak the scope from "browser engine for SerenityOS" to "cross-platform browser engine" and build something that many more people could potentially have use for some day. :^)

Note that [LibWeb started back in 2019, then called LibHTML](https://github.com/SerenityOS/serenity/commit/a67e823838943b31fb7cea68bd592093e197cf16):

```
commit a67e823838943b31fb7cea68bd592093e197cf16
Author: Andreas Kling
Date:   Sat Jun 15 18:55:47 2019 +0200

    LibHTML: Start working on a simple HTML library.
```

[LibJS began almost 9 months later, in 2020](https://github.com/SerenityOS/serenity/commit/f5476be702009968468731df5e23cdeb68fdb6e0):

```
commit f5476be702009968468731df5e23cdeb68fdb6e0
Author: Andreas Kling
Date:   Sat Mar 7 19:42:11 2020 +0100

    LibJS: Start building a JavaScript engine for SerenityOS :^)
```

If you're interested, you can see exactly how I started the LibJS engine,
as the whole process was [recorded for YouTube](https://www.youtube.com/watch?v=byNwCHc_IIM) :^)

## Basic architecture

Both LibWeb and LibJS are novel engines. I have a personal history with the Qt and WebKit projects, so there's some inspiration from them throughout, but all the code is new. Not to mention, hundreds of people have worked on the codebase since I started it, all adding their own personal influences, so it's definitely its own thing.

The browser and libraries are all written in C++. (While our own memory-safe [Jakt](https://awesomekling.github.io/Memory-safety-for-SerenityOS/) language is in heavy development, it's not yet ready for use in Ladybird.)

Here's a rough breakdown of the current stack:

- **[Ladybird](https://github.com/SerenityOS/ladybird)**: Tabbed browser GUI application
- **[LibWeb](https://github.com/SerenityOS/serenity/tree/master/Userland/Libraries/LibWeb)**: Web engine, multiple standards: HTML, DOM, CSS, SVG, ...
- **[LibJS](https://github.com/SerenityOS/serenity/tree/master/Userland/Libraries/LibJS)**: The ECMAScript language, runtime library, garbage collector
- **[LibGfx](https://github.com/SerenityOS/serenity/tree/master/Userland/Libraries/LibGfx)**: 2D graphics, text rendering, image formats (PNG, JPG, GIF, ...)
- **[LibRegex](https://github.com/SerenityOS/serenity/tree/master/Userland/Libraries/LibRegex)**: Regular expression engine
- **[LibXML](https://github.com/SerenityOS/serenity/tree/master/Userland/Libraries/LibXML)**: XML parser
- **[LibWasm](https://github.com/SerenityOS/serenity/tree/master/Userland/Libraries/LibWasm)**: WebAssembly parser and interpreter
- **[LibUnicode](https://github.com/SerenityOS/serenity/tree/master/Userland/Libraries/LibUnicode)**: Unicode support library
- **[LibTextCodec](https://github.com/SerenityOS/serenity/tree/master/Userland/Libraries/LibTextCodec)**: Text encoding conversion library
- **[LibMarkdown](https://github.com/SerenityOS/serenity/tree/master/Userland/Libraries/LibMarkdown)**: Markdown parser
- **[LibCore](https://github.com/SerenityOS/serenity/tree/master/Userland/Libraries/LibCore)**: Miscellaneous support functions (I/O, datetime, MIME data, ...)
- **[Qt](https://www.qt.io)**: Cross-platform GUI and networking

LibWeb has a `Platform` layer where Ladybird injects Qt support code for event loops, timers, system font settings, etc. We currently use Qt for networking in Ladybird, as the multi-process RequestServer system is not available outside of SerenityOS yet. Likewise, Ladybird is currently single-process, while the SerenityOS browser is process-per-tab. All of this is temporary and will change over time.

## License & "business model"

Ladybird and its engine are freely available under the [2-clause BSD license](https://opensource.org/licenses/BSD-2-Clause). You cannot buy influence over the project, but you can improve the browser by participating in development!

I'm personally working on these projects [full time since 2021](https://awesomekling.github.io/I-quit-my-job-to-focus-on-SerenityOS-full-time/) thanks to my generous supporters. If you like what I'm doing, you can help me do more of it by supporting me on [GitHub Sponsors](https://github.com/sponsors/awesomekling), [Patreon](https://patreon.com/serenityos) or [PayPal](https://paypal.me/awesomekling).

I would *love* to have enough money to pay others to work on Ladybird some day. At the moment, I make just enough to support my own family, but if things should grow past the point where I'm comfortable, I will look into restructuring so I can hire more help.

In addition to myself, you can already directly sponsor [Linus Groh](https://github.com/sponsors/linusg) and [Sam Atkins](https://github.com/sponsors/AtkinsSJ) who both do a lot of excellent browser work.

## A note on maturity

Please note that we're still early in development, and many web platform features are missing or broken. It's going to take a long time before Ladybird is ready for day-to-day browsing.

We're very much in the "make it work" part of the "make it work, make it good, make it faster" cycle. As such, we tend to focus a lot more on correctness and feature support rather than optimization. Performance work happens mostly at the architectural level, although targeted optimizations that relieve particular pain points do also happen.

Please note that this is *not* a product announcement or release, but more of a personal announcement that I'm adding *"a truly independent cross-platform browser"* to my list of personal goals. It's also an invitation to anyone who might be interested in working on a completely new browser. :^)

![Acid3 passes in Ladybird]({{ site.url }}/assets/ladybird-acid3.png)

As you can see above, we do pass the classic [Acid3 standards test](https://en.wikipedia.org/wiki/Acid3), which covers a bunch of basic CSS layout features, and various DOM/HTML APIs. However, the test does not cover many of the features used on the web today (like CSS flexbox, CSS grid, etc.)

Fidelity of modern websites in Ladybird is steadily improving, but you'll often see lots of layout and compatibility issues. For example, here's Reddit right now:

![/r/programming in Ladybird]({{ site.url }}/assets/ladybird-proggit.png)

## Community acknowledgment

Sometimes people write articles saying I'm "single-handedly" doing this or that. I'm not! Both Ladybird and SerenityOS are community efforts that hundreds of awesome people are pouring their heart and soul into. I write a lot of code, and I do my best to cheerlead for the community, but I absolutely wouldn't be here without them!

## FAQ

#### **Q: Which platforms will Ladybird support?**

So far, we've seen it running on Linux, macOS, Windows (WSL) and Android. The Linux version is definitely the most tested.

Since the libraries come from SerenityOS, they're already self-contained, and we only need Qt to help us with GUI and networking. This makes the browser quite portable, and in *theory* we could run wherever Qt runs. In practice, we'll see what happens.

#### **Q: When will Ladybird be ready for use?**

I don't know. It depends on what you consider "ready", but I'd expect a few more years of development before we have something solid. You can accelerate this process by participating in development and/or supporting our developers financially.

#### **Q: How can I participate in development?**

The most important work ahead of us is fixing bugs and adding missing features to LibWeb and LibJS. If you try opening your favorite website in Ladybird, you will find bugs! To participate in development, figure out the bug and fix it. Development discussion primarily happens in the `#browser` and `#js` channels on [our Discord server](https://discord.gg/serenityos), so come join us there.

#### **Q: I found a website that does not work! Where do I report this?**

At this point, there are *way* more websites that don't work than websites that do. We're not yet at the point where reporting individual site issues makes sense.

That said, if you're going to actually work on fixing the problems, feel free to track them using GitHub issues if that helps you.

#### **Q: Do you have a JavaScript JIT compiler?**

No, we have a traditional AST interpreter that is being replaced by a bytecode VM. You can track the [LibJS test262 score for both backends here](https://libjs.dev/test262/). I'm not convinced that the complexity and security burdens of a JavaScript JIT are reasonable, and given recent developments like Microsoft Edge's [Super Duper Secure Mode](https://microsoftedge.github.io/edgevr/posts/Super-Duper-Secure-Mode/), I'm interested in pushing for best-effort JIT-less performance while keeping the codebase simple.

#### **Q: I opened Acid3 in my browser and I only got 97/100. What's wrong?**

You're using an older version of the test that does not reflect settled web standards. The up-to-date version is here: [http://wpt.live/acid/acid3/test.html](http://wpt.live/acid/acid3/test.html).

#### **Q: Why bother? You can't make a new browser engine without billions of dollars and hundreds of staff.**

Sure you can. Don't listen to armchair defeatists who never worked on a browser.

## Contact

If you're interested in working on Ladybird, LibWeb, LibJS, or any part of the supporting stack, you can find us on the [SerenityOS Discord](https://discord.gg/serenityos). :^)
