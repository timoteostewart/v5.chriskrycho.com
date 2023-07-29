---
title: Unsafe
subtitle: >
  On a common meme: “Rust would not help because we would have to use a lot of `unsafe`”.

summary: >
  I semi-regularly hear developers who claim that using Rust would not gain them anything because they would have to use `unsafe`. But this is not true. Local reasoning matters.

tags:
  - software development
  - Rust
  - Val
  - Vala
  - Swift

qualifiers:
  audience: >
    People at least vaguely familiar with the tradeoffs around memory safety in “systems” languages like Rust, C++, Zig, Odin, etc.; and 

started: 2023-07-29T10:05:00-0600
draft: true

---

I semi-regularly hear from developers who claim that using Rust for their systems-level code would not gain them anything because they would have to use `unsafe` extensively in their code base. But this is not true.

Even in a code base which makes extensive use of Rust’s `unsafe` keyword and features, including for <abbr title="foreign function interface">FFI</abbr>, low-level bit-twiddling, and so on, the ability to distinguish between safe and unsafe code comes with non-trivial benefits.

{% note %}

This is not a “Rust is always the best” manifesto. Rather, this is a critique of one very specific misunderstanding I see floating around—even from some really sharp developers. Please read it as such, and particularly note my closing comment below!

{% endnote %}

First and foremost, it allows us to provide genuinely safe <abbr title="application programming interface">API</abbr>s which wrap the unsafe code. Those safe wrappers both *uphold* and *isolate* the key invariants for memory safety. Both of these are important, and they are closely related to each other.

When dealing in unsafe code—whether in Rust `unsafe` blocks or in all the code in languages like C and C++[^modern-cpp]—the responsibility falls to the programmer to write code which is still safe. This is possible! It is simply very difficult. They key is that if we want to have memory safety, some place in our code must actually *check* that safety. Best of all is when that can be done by the compiler (think of bounds checks on arrays/vectors &c.). A close second is explicitly encoding those checks into the types in our system ([Parse, Don’t Validate][pdv]). A third runner-up is dynamically checking the invariants at runtime and crashing noisily and eagerly if they are violated: this is always better than the kinds of problems that come from memory corruption.[^cycle-time] A safe wrapper around an unsafe <abbr title="application programming interface">API</abbr> can follow either of the latter two approaches, and both are significant improvements over *not* explicitly upholding the contract.

[pdv]: https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/

Having only one place in the code base which must uphold a given invariant means it is far easier to test and to debug when there are failures. It means the code base does not rely on people fully internalizing the rules for each <abbr title="application programming interface">API</abbr> by reading all of its comments (and those comments being correct and exhaustive!) and then being sufficiently careful everywhere they use that <abbr title="application programming interface">API</abbr>. “Don’t Repeat Yourself” is most important, and most applicable, when it comes to upholding the invariants in a codebase.

The dynamic here is similar to providing a pure functional interface in a language which is implemented with mutable data under the hood (a very common pattern in languages like OCaml and F^♯^). Mutating a new array in place can be far more efficient than using even a well-optimized persistent data structure, but providing a purely functional interface means callers do not have to care about the implementation details and still get the benefits of referential transparency.

Second, the *rest* of the codebase can still be known to be safe! Assume 70% of your code base (an incredibly high amount!) of your code base is wrapped in `unsafe`. That still means 30% of your code which can be statically known to be memory safe. This is no small thing; C, C++, Zig, Odin, etc. offer no such guarantees.[^improved] When trying to learn a system, and especially when trying to understand where things went *wrong* in a system, it is incredibly helpful to be able to know where you should start looking—and where you do not have to waste time looking.

<aside>

In practice, I cannot imagine any idiomatic code base where you would end up with those kind of ratio of safe-to-`unsafe` code. The practice of wrapping `unsafe` code in safe abstractions is a learned habit, though, so I can see how people who have not internalized it could fairly readily end up there.

</aside>

Sometimes people do manage memory safety mostly-effectively by (a) being the only person, or one of a *very* small number of people—likely not more than 3—, working on something, (b) just keeping the whole program in their head at all times, and (c) having incredibly extensive test suites. While maybe *just* possible in those contexts, this is very difficult to sustain over time: our ability to keep a program in our head degrades both as the program grows and with any time spent away from it, and providing that same level of understanding to another person is [difficult at best][naur].

[naur]: https://cdn.chriskrycho.com/file/chriskrycho-com/resources/naur1985programming.pdf

Net: `unsafe` enables you to decrease the surface area you suspect as the source of memory safety bugs when they inevitably happen. It thus improves your ability to reason *locally*: When in non-`unsafe` blocks, you do not have to think about those issues. When in `unsafe` blocks you *do*, but with a clear idea of where the boundary is.

Unless *every line* of the program in wrapped in `unsafe` (which would be extraordinary!), Rust’s distinction between safe and unsafe code is still valuable. Granted that Rust’s memory safety benefits do not come for free! There remains no free lunch. That is quite different than asserting that there is no benefit to reasoning or to verifiability associated with those costs.

There may be other ways to achieve Rust’s goals of memory safety with lower cognitive load than Rust imposes. If so, that will be great, because Rust’s cognitive load is *not low*. [Val][val] and [Vale][vale] (no relation to each other!) both look interesting here, for example, and so does [Swift]()’s still <abbr title="">WIP</abbr> [ownership system][swift-ownership]. I do not take Rust to be *remotely* the final word in this space: it is the *first* systems language be successful at industrial scales with memory safety and hopefully not the last!

[val]: https://www.val-lang.dev
[vale]: https://vale.dev
[swift-ownership]: https://github.com/apple/swift/blob/main/docs/OwnershipManifesto.md


[^modern-cpp]: This is still true in modern C++, even with tools like unique pointers, precisely because there is no way to exhaustively check the codebase for all possible variations of safety invariant violations. You cannot search for `unsafe`!

[^cycle-time]: Notice also that the first two can be caught *much* earlier than the third, because they can be caught by a compiler and do not require running the code. This is a good reason to push these kinds of invariants into a type system where possible; it brings it into [an earlier and *faster* feedback time scale][time-scale].

[time-scale]: https://v4.chriskrycho.com/2018/scales-of-feedback-time-in-software-development.html

[^improved]: Zig and Odin make some real improvements on C and C++; but they ultimately allow comparable degrees of memory unsafety.