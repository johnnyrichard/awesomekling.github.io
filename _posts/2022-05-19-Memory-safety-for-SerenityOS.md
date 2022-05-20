---
layout: post
title: "Memory safety for SerenityOS"
---

This post describes how we're going to achieve memory safety in SerenityOS.

---

After visiting my nephews for easter, I spent the drive back home thinking about the future they will grow up in. What will their computers look like? What kind of software will they use? Will any of my code still be running?

When I started the [SerenityOS](https://github.com/SerenityOS/serenity) project in 2018, I used C++ for everything, simply because it was the language I was most comfortable with. It was the right choice at the time, as it allowed me to bootstrap the project (and a community) very quickly and efficiently.

In the time since then, SerenityOS has grown larger and more complex, and we recently passed 700 individual contributors! It's *far* from a one-man hobby project at this point.

When thinking about the future, I would love for SerenityOS to be around in 30 years, when my nephews are my age (and I'm an old greybeard!)

While I believe our community and system architectures are strong enough to sustain development for years to come, I no longer believe that C++ is the right language for us.

As much as I enjoy using it, the lack of memory safety in C++ means that we'll always have bugs that could have been avoided. I'm tired of this, and I don't want to spend the next decades of my life debugging more avoidable bugs.

To improve the longevity of SerenityOS, we need to make the system memory-safe.

## So how do we achieve memory safety?

I spent a few weeks exploring the current landscape of memory-safe systems languages. I learned a handful of new ones so I could see how they work, and understand what they do to achieve safety.

I tried rewriting parts of SerenityOS in different languages, and while there were some interesting options, they all came with idiosyncratic limitations and dependencies that made them unsuitable for adoption.

Since the beginning, SerenityOS has been about making everything ourselves, for fun, for love of programming, for control, for performance, for vertical integration, etc.

In fact, the main thing we haven't made ourselves is a programming language.

## Yak-baiting a friend

Throughout this process, I'd been talking to [my friend JT](https://twitter.com/jntrnr) and sharing the struggle I had with the various languages. After talking their ear off about why some language wasn't a good fit, I got this intriguing message:

![JT saying "So... I'm totally yak baited at this point"]({{ site.url }}/assets/jt_yakbait.png)

<font size=2><b>NOTE: "yakbait" is <a href="https://github.com/SerenityOS/yaksplained#yakbait-nerd-snipe">SerenityOS slang</a> for baiting someone into shaving your yak</b></font>

JT went on to suggest that instead of settling for an existing language, we could design a new language by simply stealing the stuff we liked from other languages, and skipping the stuff we didn't like or need.

To simplify incremental adoption, the new language would transpile to C++, which could then easily interact with our existing code.

I was hooked. *Transpiling to C++!?* I didn't even realize that was an option!

We decided to name it **Jakt** ([Swedish for "hunt"](https://en.wiktionary.org/wiki/jakt#Swedish)). What followed was 2 weeks of intense compiler bootstrapping with JT.

The language is now at a point where I feel comfortable telling you that we're working on it, but it's still a ***long*** way from "ready".

## Jakt

So, let's take a look at **Jakt**! It's...

- Object-oriented
- Safe by default
- Paranoid about integer overflow & truncation
- Immutable by default
- *Young and immature, not ready for anything serious*

The current Jakt compiler is written in Rust and spits out C++. Development happens in the [jakt](https://github.com/SerenityOS/jakt) repository on GitHub.

### A little Jakt program

```
class Language {
    name: String
    age_in_days: i64
    
    function greet(this) {
        println("Hello from {}!", this.name)
        println("I am this many days old:")
        for i in 0..this.age_in_days {
            println(":^)")
        }
    }
}

function main() {
    let jakt = Language(name: "Jakt", age_in_days: 14)
    jakt.greet()
}
```

### Memory safety

So how does Jakt achieve memory safety? Through a combination of these techniques:

- Automatic reference counting of all `class` instances.
- Runtime bounds checking of arrays and slices.
- No dereferencing raw pointers in safe (default) code. `unsafe` keyword for situations where it's necessary.
- `weak` references that get emptied on pointee destruction.

## Roadmap

Here's a *very* fluffy 10 year roadmap for this project:

* Bring up basic language features. 
* Improve static analysis in the compiler.
* Write an auto-formatter for Jakt.
* Rewrite the Jakt compiler in Jakt.
* Integrate Jakt with the SerenityOS build system.
* Incrementally rewrite SerenityOS in Jakt.
* Stop transpiling to C++ and generate native code directly.

## FAQ

#### **Q: Why not just use an existing language?**

I have nothing bad to say about other languages. This is simply the option that makes the most sense for SerenityOS, which is fundamentally about having fun and implementing everything ourselves.

#### **Q: Why does Jakt have flaws?**

It's two weeks old.

#### **Q: When will Jakt be done? Will it hit 1.0?**

It will evolve with SerenityOS, so as SerenityOS matures no doubt Jakt will as well.

#### **Q: Why ARC (automatic reference counting) instead of a borrow checker?**

ARC allows the language to feel lightweight without constantly asking the user to make decisions about memory management.

There's a little bit of overhead from maintaining reference counts, but we're betting that the comfort gained will outweigh the cost.

#### **Q: What about iterator invalidation?**

We're discussing a number of approaches, but have not settled on one yet.

#### **Q: What about thread safety?**

Jakt currently does nothing to enforce thread safety. We have not started looking at this area yet.

## Contact

If you're interested in working on Jakt, you can find us on the [SerenityOS Discord](https://discord.gg/serenityos). :^)
