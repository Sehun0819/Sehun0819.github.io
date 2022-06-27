---
title: "AFL++ tutorial: How to crash Rescript parser"
categories:
  - Blog
tags:
  - AFL
  - AFL++
  - fuzzing
  - Rescript
---
# AFL++ tutorial: How to crash Rescript parser

This post is a tutorial of [AFL++](https://aflplus.plus/)[^aflpp], a popular fuzzer which has detected dozens of software bugs. In this post, the main target of fuzzing is a [parser](https://github.com/rescript-lang/syntax) of [Rescript](https://rescript-lang.org/) compiler. Motivated by [this](https://forum.rescript-lang.org/t/compiler-testing-via-fuzzing/3306/3), implying there might be bugs which can be caught with primary fuzzing techniques, I started my first bug hunt hoping it would help this young language system to be more mature. 🙂

## Background

This section explains basic concept of evolutionary fuzzing, so that make it easy to understand the intention of each step in fuzzing(label). If you don't mind, you can skip this and go to Setup(label) directly.

<figure>
  <img
  src="evolutionary_fuzzing.png"
  alt="abcde">
  <center><figcaption> Figure 1. Evolutionary Fuzzing </figcaption></center>
</figure>

Figure 1 is a brief summary of evolutionary fuzzing. 'Evolutionary' means, the fuzzer evolves by updating input corpus so that it can start another iteration based on known start point. An input is regarded as useful if it found a new coverage, and added to input corpus. For each step,

1. Fuzzing starts with user provided inputs, which comply with proper input type of PUT(program under test). In our case, they should be Rescript source files.
2. Select one input from input corpus, and randomly mutate bytes.
3. Execute PUT with mutated input.
4. If PUT crashes, detect a bug by tracing PUT with given input.
5. If mutated input explores new coverage, add it to input corpus.

Then, to recognize whether an input explored previously uncovered code, we should know what part of source code is explored during program execution with given input. To do that, the source code of PUT should be recompiled with instrumentation. That is, inserting extra code to target source in order that program execution outputs additional coverage information. Many languages are providing compilers which instrument in compile time, including Ocaml.

## Setup

There are 3 components to be installed. FYI, I've been using Ubuntu 20.04.
- [Install AFL++](#install-afl)
- [Install (AFL version of) Ocaml compiler](#install-afl-version-of-ocaml-compiler)
- [Install Rescript parser](#install-rescript-parser)

#### Install AFL++

- You can build AFL++ from source([document](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/INSTALL.md)).
- Alternatively, you can install AFL++ using apt-get or apt([instruction](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/INSTALL.md)).

#### Install (AFL version of) Ocaml compiler 

1. Install Opam, the Ocaml package manager([instruction](https://opam.ocaml.org/doc/1.1/Quick_Install.html)).
2. Find available AFL version of Ocaml compiler by command below.
   ```sh
   opam swith list-available
   ```
   I chose `4.11.2+afl` version.
3. Create switch by command below.
   ```sh
   opam switch create <switch-name> 4.11.2+afl
   ```
4. Update shell environment.
   ```sh
   eval $(opam env)
   ```

#### Install Rescript parser

1. Install prerequisite opam packages.
   ```sh
   opam install dune reanalyze
   ```
2. Install Rescript parser([instruction](https://github.com/rescript-lang/syntax#setup--usage-for-repo-devs-only)).

If the installation completed successfully, The parser binary, which is the main target of fuzzing, would be located in `<path to syntax>/_build/default/cli/res_cli.exe`.

## Fuzz

[^aflpp]: A community-maintained version of [AFL](https://lcamtuf.coredump.cx/afl/), which equips additional features and enhancements.