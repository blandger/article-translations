Prefer Rust to C/C++ for new code.
==================================

2019-02-07

-   When to use Rust
-   When to not use Rust
-   When to use C/C++
-   False reasons to use C/C++
-   Appendix: my history with C/C++
-   Appendix: the choir


*This is a position paper that I originally circulated inside the firmware community at [X](https://x.company/). I've gotten requests for a public link, so I've cleaned it up and posted it here. This is, obviously, my personal opinion. Please read the whole thing before sending me angry emails.*

**tl;dr:** C/C++ have enough design flaws, and the alternative tools are in good enough shape, that I do not recommend using C/C++ for new development except in extenuating circumstances. In situations where you actually need the power of C/C++, use [Rust](https://rust-lang.org/) instead. In other situations, you shouldn't have been using C/C++ anyway — use nearly anything else.

When to use Rust
----------------

Applications like safety-critical firmware, operating system kernels, cryptography, network protocol stacks, and multimedia decoders have (for the past 30 years or so) been primarily written in C and C++. These are also *exactly* the sorts of areas where we can't afford to be riddled with potentially exploitable bugs, like buffer overflows, dangling pointers, race conditions, integer overflows, and the like. 

Make no mistake: we treat bugs like these as though they're the cost of doing business in software, when in fact they are *specifically encouraged by design flaws in the C-family of languages.* Programs written in other languages simply *do not have* some or all of these bugs.


We used C/C++ in these application areas, rather than another language, primarily for the following reasons:

-   We can precisely control the use of memory and allocation.
-   We can access machine features with intrinsics or inline assembler.
-   We need very high performance, close to the machine's theoretical max.
-   We need to operate without a runtime, or possibly without an operating system.

Rust meets all of these criteria, but also removes the “[footguns](https://en.wiktionary.org/wiki/footgun)”: it can eliminate the majority of the security, crash, and concurrency bugs found in C-based software.

(If you *don’t* need all those criteria...well, see the next section.) 

I've been following Rust closely since 2013, and it has matured significantly. As of the end of 2018^[1](http://cliffle.com/blog/prefer-rust/#whynow)^ I think it's mature enough to start relying on, if your organization has some tolerance for the occasional sub-optimal code generation. I was an early adopter of C++11 back in 2011, and the current Rust experience is *better* than the C++11 GCC experience was at that time. Which is saying something.


Why 2018? Because it's now possible to do bare-metal and embedded development (i.e. kernel hacking) without relying on unstable features in the nightly toolchain. Plus, the [2018 Edition](https://doc.rust-lang.org/edition-guide/rust-2018/index.html) changes are excellent.

I’m putting my money where my mouth is by porting [my high-performance embedded and graphics demo code](https://github.com/cbiffle/m4vga-rs/) from C++ into Rust. This is hard-real-time code where individual CPU cycles matter, we don't have anywhere near enough RAM for the task at hand, and we're pushing the hardware to the limit. The Rust version of the code is more reliable, *often faster*, and *always shorter.*


When to *not* use Rust
----------------------

Rust excels in the spaces where C/C++ have historically reigned, but as a result, Rust requires you to think about some of the same topics as C/C++. Specifically, you will spend time considering memory allocation strategies. For most applications in 2019, this is wasted effort; just throw a garbage collector at the problem and be done with it. If you don’t need Rust’s precise control over things like memory locality and determinism, you have *a wealth of options.* 

Concrete example: if I were called upon to write a symbolic algebra evaluator, or a concurrent persistent data structure, or anything else that does heavy graph manipulation, I would probably reach for something that has a tracing garbage collector — so something other than Rust. But that something *would not be C++*, where I’d have to work just as hard as in Rust but for less payoff. 
I would personally nudge you toward Swift^[2](http://cliffle.com/blog/prefer-rust/#swift)^, but Go, Typescript, Python, and even Kotlin/Java are perfectly reasonable choices.


Last I checked, Swift doesn't have a tracing garbage collector, but its automatic memory management is clever enough that you can *almost always* pretend that it does.


When to use C/C++
-----------------

Here are some good reasons why you might choose to use C/C++ anyway:

-   You are confident that your code will never be attacked, exposed to corrupted data, or relied upon by anyone. Say, an Arduino prototyping hack. Go for it.

-   You have regulatory or contractual requirements to use a particular language. Though in this case you're likely to be writing Ada, which is significantly less bug-prone than C in the first place.

-   Your target platform does not have Rust support. Because [Rust supports nearly anything with an LLVM backend](https://forge.rust-lang.org/platform-support.html), including a bunch of platforms that aren't supported by GCC, this is a pretty short list, but it currently includes (say) the 68HC11 and 68000. (Rust is supported on MSP430, Cortex-M, etc. and AVR support is in progress. And if you're on a phone, desktop, or server, you're supported. Even on System 390.)

-   You expect your compiler/toolchain to come with a commercial support agreement. I'm not aware of anyone offering this for the Rust toolchain. I’m also not aware of anyone offering it for GCC now that CodeSourcery got bought.

-   You expect your system to get large enough that rustc's performance will become a problem for you, and you expect this to happen faster than rustc can improve. Rustc is slower at compiling than GCC is. [The team is rigorously monitoring this](https://perf.rust-lang.org/) and it has been improving. Your experience will very much depend on the complexity of your C++ code; [one of my projects](https://github.com/cbiffle/rtiow-rust/) builds faster in Rust than with GCC.

-   You have a large C++ codebase that *only* exports a C++ interface, and not a language-independent API (like an `extern "C"` interface, pipes, or RPC). C++'s semantics are so hairy that no language does a    good job interfacing with it. (Swift arguably comes the closest.) Having a system like this is going to bite you at some point.


False reasons to use C/C++
--------------------------

Here are some reasons I've heard that I believe to be leading people down a false path.

### C/C++ have 30+ years of compiler work behind them, so they'll be faster/more robust. {#c-c-have-30-years-of-compiler-work-behind-them-so-they-ll-be-faster-more-robust}

I mostly hear this from people who have not worked on compilers. It's a misconception.

C and C++ compilers have gotten dramatically better over the past few decades, not because we've gradually developed special insights into compiling C — a language which was designed to be nearly trivial to compile — but because *we've gotten better at writing compilers.*

Rust shares the same compiler backend, optimizers, and code generators as Swift and C++ (Clang) use. In most cases the code is as fast as — or faster than — compiled C/C++ *today.*


### But I have a team of well-trained C/C++ programmers who don't have time to learn a new language. {#but-i-have-a-team-of-well-trained-c-c-programmers-who-don-t-have-time-to-learn-a-new-language}

...so I have bad news for you. Your C/C++ programmers are likely not as well-trained as you think they are. I work at a dyed-in-the-wool C++ shop with some of the best programmers on the planet, and yet in code review, I still *regularly* catch them making bounds errors or [relying on undefined behavior](https://bugs.chromium.org/p/nativeclient/issues/detail?id=245). Errors they wouldn't make in Rust.


The list of people who can write C/C++ correctly, under pressure, and then keep it correct under maintenance, is *very short.* You can either invest a large *continuous* amount of effort in static analysis tools, code review, and training, or you can invest effort in teaching people a new language *today* and reinvesting that ongoing effort elsewhere.

### But I tried using Rust and I couldn't do [thing] and therefore it's a toy language. {#but-i-tried-using-rust-and-i-couldn-t-do-thing-and-therefore-it-s-a-toy-language}

For C programmers, the *thing* that they try is often an [intrusive doubly linked list](http://citeseerx.ist.psu.edu/viewdoc/download;jsessionid=99FD5852BD7E80DD0CB20F2F003CC1E7?doi=10.1.1.72.6146&rep=rep1&type=pdf), which happens to be impossible to express in *safe* Rust. (Currently. It's being worked on.) This is a common enough complaint that there is an entire tutorial based around it, [Learning Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/).

They are also [very difficult to get right in C/C++](https://www.codeofhonor.com/blog/avoiding-game-crashes-related-to-linked-lists), and I can virtually guarantee that you've written one that is simply incorrect in a threaded/SMP environment. This is *why* they're hard to express in Rust.

Some things are hard in some languages and easier in others; for example, in C++, it's really hard for me to make an existing class I don't control implement a new virtual interface, while in Rust that's trivial. That doesn't make either language a toy — it just means we'll use different solutions in each language.


Appendix: my history with C/C++
-------------------------------

I am not some guy who tried C++ and thought it was hard. I went the long way to get here.

I’ve been using C since about 1993, and C++ since 2002, both more or less continuously. I've used them in environments including Google production, Qt, Chrome, graphics demos, OS kernels, and deeply embedded battery management chips. When starting [Loon](https://loon.com/)'s firmware organization, I advocated *hard* for C++ (over C99); we migrated to C++11 bleeding-edge-early (in 2011) and *holy crap did it pay off*.  I later put a lot of energy into talking other teams at X into using C++, rather than straight C, for their firmware.


When Loon couldn't find a working bare-metal C++ `crt0.o` for the then-new Cortex-M processors, I wrote one; they're still flying it. I’ve written a [replacement for the C++ standard library](https://github.com/cbiffle/etl/) that eliminates heap allocation and adds some Rust-like features. I know the C++ standard unusually well... or at least I *did*, I've gotten rusty in the past year or two (pun intended).

In short: you would expect that my bugs-per-line-of-code rate would be pretty low by industry standards — and yet *I still produce bugs in C++ more often than I can accept.* My breakup with C++ has been slow and painful, but it feels good to finally talk about it. It’s not me, it’s you.


Appendix: the choir
-------------------

In which I compile instances of smart people agreeing with me. :-)

Chris Palmer’s [State of Software Security 2019](https://noncombatant.org/2019/01/06/state-of-security-2019/): (emphasis added)

> C++ continues to be untenably complex and wildly unsafe … I can’t possibly select and link to a list of the infinite bug reports whose root causes are memory unsafety. … In particular, **nobody should start a new project in C++.**

Alex Gaynor’s essay [The Internet Has a Huge C/C++ Problem and Developers Don't Want to Deal With It](https://motherboard.vice.com/en_us/article/a3mgxb/the-internet-has-a-huge-cc-problem-and-developers-dont-want-to-deal-with-it) (in Vice, of all places):

> Memory unsafety is currently a scourge for our industry. But it doesn't have to be the case that every Windows or Firefox release fixes dozens of avoidable security vulnerabilities. We need to shift ourselves from treating each memory unsafety vulnerability as an isolated incident, and instead treat them as the deeply rooted systemic problem they are. And then we need to invest in engineering research into how we can build better tools to solve this problem.

(He’s also got some [great blog posts](https://alexgaynor.net/2019/apr/21/modern-c++-wont-save-us/) on the topic.)

Manish Goregaokar, from Mozilla, [notes on ycombinator](https://news.ycombinator.com/item?id=15350282) that fuzzing the Rust parts of Firefox does not find safety bugs in the Rust code, but does find bugs in the C++ code it replaces:

> > Happily, not one of those bugs could actually be escalated into an actual exploit. In each case, Rust's various runtime checks successfully caught the problem and turned it into a controlled panic. 

> This has been more or less our experience with fuzzing rust code in firefox too, fwiw. Fuzzing found a lot of panics (and debug assertions / "safe" overflow assertions). In one case it actually found a bug that  had been under the radar in the analogous Gecko code for around a decade.


Copyright ©2011-2019 Cliff L. Biffle —
[Contact](http://cliffle.com/contact/)
