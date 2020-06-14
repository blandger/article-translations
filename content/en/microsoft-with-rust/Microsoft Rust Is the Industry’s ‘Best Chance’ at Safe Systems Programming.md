2020-06-10 10:10:58

Microsoft: Rust Is the Industry’s ‘Best Chance’ at Safe Systems Programming
===========================================================================

#### 10 Jun 2020 10:10am, by [Joab Jackson](https://thenewstack.io/author/joab/ "Posts by Joab Jackson")

![](https://cdn.thenewstack.io/media/2020/06/1a99000f-microsoft-rust.jpg)


No matter how much investment software companies may put into tooling and training their developers, “C++, at its core, is not a safe language,” said [Ryan Levick](https://github.com/rylev), [Microsoft](https://www.microsoft.com) cloud developer advocate, during the AllThingsOpen virtual conference last month, explaining, [in a virtual talk](https://youtu.be/NQBVUjdkLAA), why Microsoft is gradually switching to Rust to build its infrastructure software, away from C/C++. And it is encouraging other software industry giants to consider the same.

“We’re using languages that are, because they are quite old and come from a different era, do not provide us the ability to protect ourselves from these kinds of vulnerabilities, he [said](https://youtu.be/NQBVUjdkLAA?t=580). “C++ is not a memory safe language and no one would really pretend that it is,” he said.

In fact, Microsoft has deemed C++ no longer acceptable for writing mission-critical software. The industry sorely needs to move to a performant, memory-safe language for its low-level system work. And the best choice on the market today is [Rust](https://www.rust-lang.org/), Levick said.

C/C++ Can’t Be Fixed
--------------------

Today, C and C++ are the go-to languages for writing core system software. It is fast, with the only assembly between the code and the machine itself.

But the industry is being crippled by all the memory-related bugs — many of which are security hazards — caused by these languages. Now, 70% of the CVEs originating at Microsoft are memory safety issues, Levick said. “There is no real trend it’s just staying the exact same,” he said. “Despite massive efforts on our part to fix this issue it still seems to be a common thing.”

From a financial perspective, it makes sense, given the soaring cost of remedying this never-ending stream of memory-related errors. Back in 2004, each memory-related error cost industry about \$250,000 each, and that Microsoft estimation is probably on the lower-end, Levick said.

Of course, there are a number of efforts to boost C++ security, but while each is effective in the way it does, none entirely solves the problem.

One approach that has been long floated is to do more programmer training in how to write safer code. But, “there is zero evidence that doing holistic training of C and C++ developers will actually fix this issue in any significant way,” Levick said, citing Microsoft’s own heaps of dev internal training.

Static analysis is cited as another possible solution. But static analysis comes with too much overhead: It needs to be wired into the build system. “So there’s a lot of incentive not to use static analysis,” Levick said.  “If it’s not on by default it won’t it won’t help.”

The same goes for runtime checks: “It’s impossible or it’s very least extremely hard to know when runtime checking contracts are used and when they’re not,” he said, adding that they also come with an operational overhead.

**The Industry’s Best Chance**
------------------------------

In response to this problem of memory-related errors, the [Microsoft Security Response Center](https://www.microsoft.com/en-us/msrc) launched the [Safe Systems Programming Language](https://www.youtube.com/watch?v=t3dKNcJXtbg) initiative. There, some work was dedicated for shoring up C/C++. [Verona](https://github.com/microsoft/verona), a new programming language being created for safe low-level programming, was also created here. But the third prong of the project strategy, the one they are putting the most faith in, is to support “the industry’s best chance for addressing this issue head-on.”

“And we believe that to be Rust,” he said.

Performance-wise, Rust is on par with C/C++, and maybe even slightly faster. Rust brings developer productivity, with package management, modern testing frameworks and the like. And programmers [love Rust](https://stackoverflow.blog/2020/01/20/what-is-rust-and-why-is-it-so-popular/) for it.

But the main reason Microsoft is so enamored with Rust is that it is a memory-safe language, one with minimal runtime checking. Rust excels in creating correct programs.  **Correctness** means, roughly, that a program is checked by the compiler for unsafe operations, resulting in fewer runtime errors. Unsafe keyword is an option, but not the default. Unsafe Rust code is almost always a subset of a larger body of safe code. Unsafe mode is necessary for memory-assigning jobs like writing device drivers. But even here the unsafe portions of memory are encapsulated behind an API.

This ability to program safely is not one that should be taken lightly, Levick said. In fact, it can provide more than a 10x improvement, making it worthwhile for investment. This is largely because pretty much all C/C++ code needs to security audits for unsafe behavior, whereas unsafe code written in Rust that would need to be checked is only a small subset of most code bases.

While Microsoft is bullish on Rust, Levick admits that Microsoft core developers won’t stop using C/C++ anytime soon.

“We have a lot of C++ at Microsoft and that code is not going anywhere,” he said. “In fact, Microsoft c++ continues to be written and will continue to be written for a while.”

A lot of tooling is built around C/C++. In particular, Microsoft binaries are now almost completely built on the Microsoft Visual C++ compiler which produces MSVC binaries, whereas Rust relies on [LLVM](https://llvm.org/).

Perhaps the biggest challenge, though, is cultural. “There is just some people that just want to get their job done in the language that they already know,” Levick admitted.

Still, the industry seems to be moving towards Rust. Amazon Web Services uses it, in part for deploying the [Lambda serverless runtime](https://aws.amazon.com/lambda/), as well as for some parts of [EC2](https://aws.amazon.com/ec2/). Facebook has [started using](https://www.youtube.com/watch?v=kylqq8pEgRs) Rust, as has Apple, Google, Dropbox and Cloudflare.

The dates All Things Open 2020 have been announced: [Oct. 20-22](https://2020.allthingsopen.org/).

*The New Stack does not allow comments directly on this website. We invite all readers who wish to discuss a story or leave a comment to visit us on Twitter or Facebook. We also welcome your news tips and feedback via email: **[feedback@thenewstack.io](mailto:feedback@thenewstack.io).*

Amazon Web Services is a sponsor of The New Stack.

#### © 2020 The New Stack. All rights reserved.

[Privacy Policy](/privacy-policy). [Terms of Use](/terms-of-use).
