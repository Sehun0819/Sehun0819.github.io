---
title: "AFL++ tutorial: How to crash ReScript parser"
date: 2022-06-30
categories:
  - Blog
permalink: /posts/fuzz-rescript/
tags:
  - AFL
  - AFL++
  - fuzzing
  - ReScript
---
# AFL++ tutorial: How to crash ReScript parser

This post is a tutorial of [AFL++](https://aflplus.plus/)[^aflpp], one of the most famous fuzzing tools which has detected dozens of software bugs. In this post, we utilize this fuzzer to crash the [parser](https://github.com/rescript-lang/syntax) of [ReScript](https://rescript-lang.org/) compiler. Motivated by [this](https://forum.rescript-lang.org/t/compiler-testing-via-fuzzing/3306/3), implying there might be bugs which can be caught with primary fuzzing techniques, I started my first bug hunt hoping it would help this young language system to be more mature.

> **Note**
> If you are busy, feel free to visit this [example repository](https://github.com/Sehun0819/fuzz-rescript-parser) and start fuzzing immediately .

## Background

This section explains basic concept of evolutionary fuzzing which is a mainstream of fuzzing technique[^fuzzingsurvey]. You can use AFL++ without reading this explanatory section, so if you don't mind how the fuzzer struggles with finding bug you can skip this and go to [Setup](#setup) directly.

![image](/images/evolutionary_fuzzing.png)

Above figure is a brief summary of evolutionary fuzzing. 'Evolutionary' means, the fuzzer evolves by updating input corpus so that it can start another iteration based on known, effective start point. An input is regarded as useful if it found a new coverage, and added to input corpus. Many state-of-the-art fuzzers including [AFL](https://lcamtuf.coredump.cx/afl/) and [LibFuzer](https://llvm.org/docs/LibFuzzer.html) have adopted this strategy. For each interation in above figure,

1. Fuzzing starts with user provided inputs, which comply with proper input type of PUT(program under test). In our case, they should be ReScript source files.
2. Select one input from input corpus, and randomly mutate in byte-level.
3. Execute PUT with the mutated input.
4. If PUT crashes, detect a bug by tracing PUT with given input.
5. If the mutated input explores new coverage, add it to input corpus.

To recognize whether an input explores previously uncovered code, we should know what part of source code is explored during program execution with given input. To do that, the source code of PUT should be recompiled with instrumentation. Instrumentation means inserting extra code in order that program execution outputs additional coverage information. Many languages are providing compilers which are equipped with instrumenting functionality for AFL, including OCaml.

## Setup

There are 3 components to be installed. FYI, I've been using Ubuntu 20.04.
- [Install AFL++](#install-afl)
- [Install (AFL version of) OCaml compiler](#install-afl-version-of-ocaml-compiler)
- [Install ReScript parser](#install-rescript-parser)

#### Install AFL++

- You can build AFL++ from source([instruction](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/INSTALL.md)).
- Alternatively, you can install AFL++ using apt-get.
  ```sh
  sudo apt-get update
  sudo apt-get -y install afl++
  ```

#### Install (AFL version of) OCaml compiler 

1. Install Opam, the OCaml package manager([instruction](https://opam.ocaml.org/doc/1.1/Quick_Install.html)).
2. Find available OCaml compiler version by command below. Versions with AFL instrumenting functionality end with `+afl`.
   ```sh
   opam swith list-available
   ```
   FYI, The latest version supporting AFL instrumentation is `4.11.2+afl`.
3. Create a switch with found OCaml compiler version by command below.
   ```sh
   opam switch create <switch-name> 4.11.2+afl
   ```
4. Update shell environment.
   ```sh
   eval $(opam env)
   ```

#### Install ReScript parser

Install ReScript parser([instruction](https://github.com/rescript-lang/syntax#setup--usage-for-repo-devs-only)). (AFL version of)OCaml compiler automatically instruments parser source codes(with out any special flag or something) so that its coverage can be measured by AFL++.
   > **Note**
   > In this post, previous commit(af00a46042c76ca8342dd6857ebe8776de00200c) is used instead of latest to reproduce this [bug](https://github.com/rescript-lang/syntax/pull/540).
   > If you want to follow, checkout to that commit before building.

Assuming that ReScript parser is cloned under `$HOME`, The path of parser binary, which is the target of fuzzing, would be `~/syntax/_build/default/cli/res_cli.exe`.

## Prepare Fuzzing Inputs

Gather valid inputs for PUT, and minimize input corpus by removing redundant inputs whose coverages are not unique. There was an empirical study[^seedselection] which demonstrated that the input seeds have a critical impact on fuzzing performance, and the result implies that starting with large & unique input seeds is the best.

#### Collect ReScript source files

Gather ReScript repositories in a directory and copy only rescript sources.
1. Make a directory `rescript_projs` under `$HOME`.
   ```sh
   cd ~
   mkdir rescript_projs
   ```
2. Clone any available ReScript repositories. You can find a variety of ReScript repos in [LibHunt](https://www.libhunt.com/l/rescript). Or, using just ReScript parser repository is a good compromise as there are quite a lot of ReScript sources including testing inputs.
   ```sh
   cd rescript_projs
   git clone git@github.com:rescript-lang/syntax.git
   # clone as many ReScript repositories as you want.
   ```
3. Copy only ReScript sources into another directory, `rs_files`.
   ```sh
   cd ~
   mkdir rs_files
   find rescript_projs -type f \( -name "*.res" -o -name "*.resi" \) | xargs cp -n -t rs_files/
   ```

#### Minimize input corpus

Gathered source files in `rs_files` should be minimized so that only unique inputs are left. As mentioned above, an input is 'unique' if its coverage is new. Intuitively, fuzzer would stick to exploring similar codes rather than searching diversely if there were lots of duplicates. This step is done by AFL++, with instrumented target program.
```sh
afl-cmin -i ~/rs_files -o ~/rs_files_unique -- ~/syntax/_build/default/cli/res_cli.exe  @@
```
This command copies only unique inputs from `rs_files` to `rs_files_unique`. You don't have to create `rs_files_unique` directory explicitly because the process automatically creates it. `@@` is a place holder where a fuzzed input is supposed to be.

You can see more than 40% of sources from ReScript parser repo were thrown away, through this minimization process.
```sh
$ ls ~/rs_files | wc -l
1069
$ ls ~/rs_files_unique | wc -l
626
```

## Do Fuzzing

Start fuzzing by typing a command below. Again, `fuzz_report`, a directory for the reports is created automatically.
```sh
afl-fuzz -i ~/rs_files_unique -o ~/fuzz_report -- ~/syntax/_build/default/cli/res_cli.exe  @@
```
You will see an interface that shows fuzzing status.

![image](/images/aflplusplus.png)

Fuzzed inputs which successfully crashed the parser can be found in `~/fuzz_report/default/crashes/`.
Below is one of fuzzed inputs made by AFL++, which crashed (older version of) ReScript parser.
```ocaml
a->f(b, c)->g(d, e)

Element.querySelectorAll(selector, element)
->NodeList.toArray
->Array.keepMap(Elemelet options: string => Warp_Types_Client.t<Warp_Types_ResponseType.payload<option<string>>>

let get: string => Warp_Types_Client.t<Warp_Types_ResponseType.payload<option<string>>>

let head: string => Warp_Types_Client.t<Warp_Types_ResponseType.payload<optio[<string>>>

let post: string => Warp_Types_Client.t<Warp_Types_Respon<9f>eType.payload<option<string>>>

let put: string => Warp_Types_Client.t<Warp_Types_ResponseType.payload<option<string>>>

let delete: string => ^B^@rp_Types_Client.t<Warp_Types_ResponseType.payload<option<string>>>

let trace: string => Warp_Types_Client.t<Warp_Types_ResponseType.payload<optio
```

You can manually crash the parser with found crashing inputs. Please note that you should set OCaml environment variable to trace a call stack.
```sh
export OCAMLRUNPARAM=b
~/syntax/_build/default/cli/res_cli.exe ~/fuzz_report/default/crashes/<fuzzed input>
```

Generated stack trace properly points out problematic source locations, which has been handled by this [PR](https://github.com/rescript-lang/syntax/pull/540).
```
Fatal error: exception Stack overflow
Raised by primitive operation at Syntax__Res_parser.expect in file "src/res_parser.ml", line 122, characters 5-20
Called from Syntax__Res_core.parseHashIdent in file "src/res_core.ml", line 676, characters 2-22
Called from Syntax__Res_core.parsePolymorphicVariantType.loop in file "src/res_core.ml", line 4978, characters 32-69
Called from Syntax__Res_core.parsePolymorphicVariantType.loop in file "src/res_core.ml", line 4979, characters 21-27
Called from Syntax__Res_core.parsePolymorphicVariantType.loop in file "src/res_core.ml", line 4979, characters 21-27
Called from Syntax__Res_core.parsePolymorphicVariantType.loop in file "src/res_core.ml", line 4979, characters 21-27
Called from Syntax__Res_core.parsePolymorphicVariantType.loop in file "src/res_core.ml", line 4979, characters 21-27
...
```

## Conclusion

AFL++ was able to find previously unknown bugs of ReScript parser. Moreover, it seems that there are still rooms for this tool to play an important role. I hope this post help ReScript developers to utilize AFL++ to make ReScript parser more robust.

The reason why this general fuzzing tool works well comes from the fact that the input type of parser is 'string', which is a simple, one-dimensional structure. Byte-level mutation strategies of AFL++ wouldn't break the input validity. On the other hand, if you want to fuzz ReScript compiler whose input must be valid AST, naive application of this tool won't work because byte-level mutation of AFL++ would probably make input source code invalid, leading to failure in parsing process. But anyway, if what all you need is to find bugs of parser, AFL++ would work for you.

[^aflpp]: A community-maintained version of [AFL](https://lcamtuf.coredump.cx/afl/), which equips additional features and enhancements.
[^fuzzingsurvey]: If you are looking for more comprehensible introduction for fuzzing, I strongly recommend this [paper](https://ieeexplore.ieee.org/document/8863940).
[^seedselection]: Herrera et al., "Seed selection for successful fuzzing". ISSTA 2021. https://doi.org/10.1145/3460319.3464795
