---
layout: blog
post-type: blog
by: Lukas Rytz
title: Performance of trait methods
---

# Performance of using default methods to compile Scala trait methods

In Scala 2.12, bodies of methods defined in traits will be compiled to default methods in the
interface classfile. In short, we have the following bytecode formats for concrete trait methods:

  - 2.11.x: trait method bodies are in static methods in the trait's `T$impl` class. Classes
    extending a trait get a virtual method that implements the abstract method in the interface and
    forwards to the static implementation method.
  - 2.12.0-M4: trait method bodies are in (non-static) interface default methods, subclasses get an
    virtual method (overridding the default method) that forwards to that default method using
    `invokespecial` (a `super` call).
  - [33e7106](https://github.com/scala/scala/commit/33e7106): in most cases, no more forwarders are
    generated in subclasses as they are not needed: the JVM will resolve the correct method.
    Concrete trait methods are invoked either using `invokeinterface` (if the static receiver type
    is the trait) or `invokevirtual` (if the static receiver type is the subclass).
  - 2.12.0-M5: trait method bodies are emitted in static methods in the interface classfile. The
    default methods forward to the static methods.

Recently we observed that 33e7106 causes a 20% slowdown of the Scala compiler (tested by compiling
the sources of [better-files](https://github.com/pathikrit/better-files)). (Since we're still
lacking a proper performance regression testing infra, the slowdown was discovered later and pinned
down using git-bisect).

First observation: the slowdown is not due to additional logic introduced by the patch, but the
change in the bytecode of the compiler itself. This can be verified easily: a compiler built from
revision 33e7106 using its parent (b932419) as STARR has no slowdown. Building it with itself as
STARR, the resulting compiler runs slower.

This means that any Scala applications using concrete trait methods is likely to be affected by
this problem.

This post logs our attempts to find the root cause of the slowdown.

## Some details on the HotSpot compiler

This section explains some details of the HotSpot optimizer. It assembles information from various
sources and own observations; it might contain mistakes and misunderstandings. It is certainly
simplified and incomplete. More details are available in the linked resources.

### JITing

My recommended reference for this first section is the talk "JVM Mechanics" by Doug Hawkins
([video](https://www.youtube.com/watch?v=E9i9NJeXGmM),
[slides](http://www.slideshare.net/dougqh/jvm-mechanics-when-does-the)).

First of all, JVM 8 uses two JIT compilers: C1 and C2. C1 is fast but performs only basic
optimizations, in particular it does not perform speculative optimizations based on profiling
(frequency of branches, type profiles at callsites). C2 is profile-guided and speculative but
slower.

The JVM starts by interpreting the program. It only compiles methods that are either called often
enough or that have long enough loops. There are two counters for each method:

  - the number of times it is invoked, and
  - the number of times a backwards branch is executed.

The decision to compile a method is based on these counters. A simplified (ignoring backwards
branches), typical scenario: after 2000 invocations a method gets compiled by C1, after 15000 it
gets re-compiled by C2 (see [this answer on SO](http://stackoverflow.com/a/35614237/248998) for more
details). Note that the C1-generated assembly is instrumented to update the two counters (and also
to collect other profiling data that will be used by the C2 optimizer). After compiling a method,
new invocations of the method will use the newly generated assembly.

The above works well for a method that is invoked many times, but what happens to a long-running
method that is invoked only once, but has a long loop? The decision to compile this method is taken
when the counter of backwards branches passes a threshold. Once compilation is done, the JVM
performs a so-called on-stack replacement (OSR): the stack frame of the running method is modified
as necessary and execution continues using the new assembly.

An OSR / loop compilation of a method is always tied to a specific loop: the entry point of the
generated assembly is at the end of the loop (locations are referred to by the index in the jvm
bytecode, called "bytecode index" / `bci`). If there are multiple hot loops within a method, the
same method may get multiple OSR compiled versions. More details on this can be found in
[this post](https://gist.github.com/rednaxelafx/1165804#osr) by Krystal Mok which explains the
many details of the `-XX:+PrintCompilation` output.

### Inlining

For this section my reference s Aleksey Shipilёv's extensive post
[The Black Magic of (Java) Method Dispatch](http://shipilev.net/blog/2015/black-magic-method-dispatch/).

Inlining is fundamental because it acts as an enabler for most other optimizations. As Aleksey says
in the conclusion: "inlining actually broadens the scope of other optimizations, and that alone is,
in many cases, enough reason to inline".

Both C1 and C2 perform inlining. The policy whether to inline a method is non-trivial and uses
several heuristics (implemented in
[bytecodeInfo.cpp](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/f22b5be95347/src/share/vm/opto/bytecodeInfo.cpp),
methods `should_inline`, `should_not_inline` and `try_to_inline`). A simplified summary:

  - Trivial methods (6 bytes by default, `MaxTrivialSize`) are always inlined.
  - Methods up to 35 bytes (`MaxInlineSize`) invoked between 1 and 250 (`MinInliningThreshold`) are
    inlined.
  - Methods up to 325 bytes (`FreqInlineSize`) are inlined if the callsite is "hot" (or "frequent"),
    which means it is invoked more than 20 times (no command-line flag in release versions) per one
    invocation of the caller method.
  - The inlining depth is limited (9 by default, `MaxInlineLevel`).
  - No inlining is performed if the callsite method is already very large.

  // this is not entirelly true. It uses multiple measures of `large`.
  // inlining is performed twice. Once, on bytecode and that's the part that you've correctly covered,
  // second time on the assembly. And size limits are different and measure different notions.
  // InlineSmallCode option guards inining of assembly.
  // neither of approaches is good:
  //  - the problem with counting size of bytecode before inlining is that it counts unreachable code&gotos
  //  - the problem with inlining assembly happens as it frequently has ridiculosly bad code(eg a lot of moving data around) in "slow paths" and
          the size of slow path contributes to the measured size of method.

The procedure is the same for C1 and C2, it uses the invocation counter that is also used for
compilation decisions (previous section).

### Inlining virtual methods

In C1, a method can only be inlined if it can be statically resolved. This is the case for static
and private methods, for constructors, but also for virtual methods that are never overridden. The
JVM has full knowledge on code of the program it is executing. If a method is virtual and could
in principle be overridden in a subclass, but no such subclass has been loaded (so far), an
invocation of the method can only resolve to that single definition.

The process of analyzing the hierarchy of classes currently loaded in the VM is called "class
hierarchy analysis" (CHA). Both C1 and C2 use the information computed by CHA to inline calls to
virtual methods that are not overridden.

When the JVM loads a new class, a virtual method that was statically not overridden by CHA may get
an override. All assembly code that made use of the now invalid assumption is discarded and the
// the text reads as if the methods are executed twice. I'd rewrite as `the next execution of method will began in intepreter`
// this leaves uncovered the question of `what happens if return address is in discarded code` but I believe it's fine.
corresponding methods are executed again by the bytecode interpreter. This process is called
deoptimization. In case a deoptimized method is currently being executed, control is passed to
the interpreter by on-stack replacement.
// AFAIK this is false... Please tell me if what I write below isn't true:
// when breaking CHA assumption, VM will ask all threads to pause at safe poins.
// JVM can recreate state of interpreter in every safe-point. If method happens to pause inside method whose code is
// being discarded it will jump into interpreter

// if the method is NOT being currenly executed, but some thread has instructuion inside it in return pointer
// the method is filed with NO-OP and a special handler in the end that is able to jump to nearest safe-point.
// AFAIK this has little if something to do with OSR.

In addition to using CHA, the C2 compiler performs speculative inlining of virtual methods based on
the type profiles gathered by the interpreter and the C1-generated assembly. If the receiver type
at a callsite is always the same (the callsite is "monomorphic") the method is inlined. The assembly
contains a type test to validate the assumption, if it breaks the method gets deoptimized.

C2 will also inline bi-morphic callsites: the code of both callees is inlined, a type test is used
to branch to the correct one (or to bail out). Finally, if the type profile shows a clear bias to
a specific receiver type (for example 90%), its method is inlined and virtual dispatch is used for
the other cases (shown in Aleksey's post).

If a callsite has 3+ receiver types without a clear bias, C2 does not inline and an ordinary method
lookup is performed at runtime.

Note that C2 performs other speculative optimizations than profile-based inlining, for example
profile-based branch elimination.

// The most important improvement that C2 has is actually a good register allocator. The one in C1 is "fast and dirty"

## Understanding the performance regression

With the above knowledge at hand (I wish I had it when I started) we try to identify what causes
the slowdown of eliminating forwarder methods.

### Call performance

In a first step we measured the call performance of the various trait encodings.

The first benchmark
[`CallPerformance`](https://github.com/lrytz/benchmarks/blob/master/src/main/java/traitEncodings/CallPerformance.java)
has roughly the following structure:

    interface I {
        default int addDefault(int a, int b) { return a + b; }

        static int addStatic(int a, int b) { return a + b; }
        default int addDefaultStatic(int a, int b) { return addStatic(a, b); }

        default int addForwarded(int a, int b) { return a + b; }

        int addInherited(int a, int b);

        int addVirtual(int a, int b);
    }

    static abstract class A implements I {
        public int addInherited(int a, int b) { return a + b; }
    }

    static class C1 extends A implements I {
        public int addForwarded(int a, int b) { return I.super.addForwarded(a, b); }

        public int addVirtual(int a, int b) { return a + b; }
    }

There are identical copies of `C1` (`C2`, ...). The example encodes the following formats (we
don't test the 2.11.x format):

  - `addDefault` for 33e7106
  - `addDefaultStatic` for 2.12.0-M5
  - `addForwarded` for 2.12.0-M4

The methods `addInherited` and `addVirtual` don't represent trait method encodings, they are for
comparison. We test all encodings in a monomorphic callsite (receiver is always `C1`) and in a
polymorphic one.

#### Monomorphic case

In the monomorphic case all trait encodings are inlined and perform the same (there are tiny
differences, if you are interested check Aleksey's blog post).

If we annotate all methods with JMH's `DONT_INLINE` directive, encodings with a forwarder (either
M4-style forwarder invoking the trait default method, or the upcoming M5-style default method
forwarding to static) are a bit slower (so a default method without a forwarder is faster).
The penalty for having either forwarder is similar.

#### Polymorphic case

If the callsite is polymorphic:

  - The M4 encoding (`addForwarded`) is slow because the forwarder cannot beinlined. This the known
    issue of trait methods leading to megamorphic callsites that exists in Scala 2.11.x and older.
  - The 33e7106 (`addDefaultStatic`) and M5 (`addDefault`) encodings are also slow: the default
    method is not inlined (checked with `-XX:+PrintInlining` and by comparing with a method marked
    `DONT_INLINE`). We will explore this in detail later.

For comparison, an invocation of `addInherited` is inlined and therefore much faster. So an
inherited virtual method is not treated in the same way as an inherited default method. The next
section goes into details why this is the case.

*Note:* for the question why the 33e7106 ecoding causes a 20% performance regression, this cannot be
the reason. We found out that `addDefault` is slower than it could be in the polymorphic case, but
it is not slower than the M4 encoding.


### CHA and default methods

The reason `addDefault` is not inlined while `addInherited` in the previous example has to do with
CHA: in fact, CHA is disabled altogether for default methods. This is logged in the JVM bugtracker
under [JDK-8036580](https://bugs.openjdk.java.net/browse/JDK-8036580). It was disabled in order to
fix [JDK-8036100](https://bugs.openjdk.java.net/browse/JDK-8036100) which lead to the wrong method
being inlined. (It was @retronym who initially suggested these tickets could be relevant).

The reason for `addInherited` being inlined is that the VM knows (from CHA) the method is not
overridden in any of the loaded classes. This is tested in the
[`InliningCHA`](https://github.com/lrytz/benchmarks/blob/master/src/main/java/traitEncodings/InliningCHA.java)
benchmark.

The first benchmark measures a megamorphic call to `addInherited`, just like in the previous
section. This call is inlined. The second benchmark performs the exact same operation but makes sure
that new subclass `CX` is loaded which overrides `addInherited`. CHA no longer returns a single
target for the method and the call is not inlined. Note that no instance of `CX` is created.

This seems to be a shortcoming in C2's inliner implementation: based on the type profiling data,
C2 knows that the only types reaching the callsites are `C1`, `C2`, `C3` and `C4`. Using CHA it
could in principle find out that there is a single implementation of `addInherited`.


### Method lookup in classes implementing many interfaces

We are still searching for an answer why 33e7106 caused a performance regression. Martin Thompson
notes in a
[blog post](http://mechanical-sympathy.blogspot.ch/2012/04/invoke-interface-optimisations.html)
(dated 2012):

  > I have observed that when a class implements multiple interfaces, with multiple methods,
  > performance can degrade significantly because the method dispatch involves a linear search of
  > method list

We can reproduce this in the benchmark
([`InterfaceManyMembers`](https://github.com/lrytz/benchmarks/blob/master/src/main/java/traitEncodings/InterfaceManyMembers.java)).
The basic example is the following:

    interface I1 { default int a1 ... }
    interface I2 { default int b1 ... ; default int b2 ... }
    ...

    class A1 implements I1 { }
    class A2 implements I2 { }
    ...

    class B1 implements I1 { }
    class B2 implements I1, I2 { }
    ...

In the benchmark, every class (`A1`, `A2`, `B1`, ...) exists in four copies to make sure the
callsite is megamorphic. We measure how much time an invocation of one default method takes:

  - The number of default methods in an interface does not matter, so `A1.a1` and `A2.b1` perform
    the same.
  - The number of implemented interfaces matters, there's a penalty for every additional interface.
    So `B1.a1` is faster than `B2.b1`, etc.

Adding an overriding forwarder method to the subclasses does not change this result, the slowdown
per additional interface remains. So this seems not to be the reason for the performance regression.

### Back to CHA

Googling a little bit more about the performance of default methods, I found a relevant
[post on SO](http://stackoverflow.com/questions/30312096/java-default-methods-is-slower-than-the-same-code-but-in-an-abstract-class)
containing a nice benchmark.

I simplified the example into the benchmark
([`NoCHAPreventsOptimization`](https://github.com/lrytz/benchmarks/blob/master/src/main/java/traitEncodings/NoCHAPreventsOptimization.java)),
which is relatively small:

    interface I {
        int getV();
        default int accessDefault() { return getV(); }
    }

    abstract class A implements I {
        public int accessVirtual() { return getV(); }
        public int accessForward(){ return I.super.accessDefault(); }
    }

    class C extends A implements I {
        public int v = 0;
        public int getV() { return v; }
    }

The benchmark shows that `c.v = x; c.accessDefault()` is 3x slower than
`c.v = x; c.accessVirtual()` or `c.v = x; c.accessForward()`.

As noted in comments on the StackOverflow thread, everything is inlined in all three benchmarks,
so the difference is not due to inlining. We can observe that the assembly generated for the
`accessDefault` case is less optimized than in the other cases. Here is the output of JMH's
`-prof perfasm` feature, it includes the assembly of the hottest regions:

  - for [accessDefault](https://gist.github.com/lrytz/f1c24e685b871639d7e618b56325e102#file-adefault-txt)
  - for [accessVirtual](https://gist.github.com/lrytz/f1c24e685b871639d7e618b56325e102#file-bvirtual-txt)
  - for [accessForward](https://gist.github.com/lrytz/f1c24e685b871639d7e618b56325e102#file-cforward-txt)

In fact, the assembly for the `accessVirtual` and `accessForward` cases is identical.

One answer on the SO thread suggests that lack of CHA for the default method case prevents
eliminating a type guard, which in turn prevents optimizations on the field write and read.
Somebody with more experience in assembly code than me could certainly verify that.

I did not do any further research to find out what kind of optimizations depend on CHA, or if it is
really the lack of CHA that causes the code not to be optimized properly. For my convenience, let's
say that's beyond the scope of this post. If you have any insights or references on this topic,
please forward them to me!

It seems that CHA preventing certain optimizations is the most likely source for the slowdowns we
notice when running the Scala compiler.

## Summary

We found a few interesting limitations in the JVM optimizer:

  - Because CHA is not supported for default methods, a megamorphic callsite to a default method is
    never inlined even if the method is not overridden at all.
  - Interface method lookup slows down by the number of interfaces a class implements.
  - While monomorphic calls to default methods are inlined, the lack of CHA has negative effects
    on other optimizations.

## References

Besides the [post](http://shipilev.net/blog/2015/black-magic-method-dispatch/) already mentioned,
Aleksey Shipilёv's [blog](http://shipilev.net/) is an excellent resource for Java and JVM
intrinsics.

The talk "JVM Mechanics" by Doug Hawkins was also mentioned above
([video](https://www.youtube.com/watch?v=E9i9NJeXGmM),
[slides](http://www.slideshare.net/dougqh/jvm-mechanics-when-does-the)),
it is a great overview on the JIT, inliner and optimizer. For an overview I can also recommend a
[longer blog post](http://middlewaresnippets.blogspot.ch/2014/11/java-virtual-machine-code-generation.html)
by René van Wijk and a
[shorter one](https://www.lmax.com/blog/staff-blogs/2016/03/05/observing-jvm-warm-effects/)
by Mark Price focussing on the JIT compilers.

The JVM has an excessive number of flags for logging and tweaking:

  - Some flags are [documented here](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)
  - Many others are not documented, run `java -XX:+PrintFlagsFinal` to get a list of all flags

Some flags used in the examples of this post:

  - `-XX:TieredStopAtLevel=1` to disable C2
  - `-XX:+PrintCompilation` logs methods being compiled (and deoptimized)
  - `-XX:+PrintInlining` logs callsites being inlined (or not), best used together with the above

[JITWatch](https://github.com/AdoptOpenJDK/jitwatch) is a GUI tool that helps understanding what the
JIT is doing (I haven't tried it yet).

A
[thread](http://mail.openjdk.java.net/pipermail/hotspot-compiler-dev/2015-April/thread.html#17649)
on the hotspot-compiler-dev mailing list on why CHA is disabled for interfaces. Seems to discuss
the situation before default methods were a common thing.

A [gist](https://gist.github.com/rednaxelafx/1165804#file-notes-md) by Krystal Mok explaining many
details of the `-XX:+PrintCompilation` output and other details of the JIT process.

The [glossary](http://openjdk.java.net/groups/hotspot/docs/HotSpotGlossary.html) on the HotSpot
wiki contains some useful nomenclature.
