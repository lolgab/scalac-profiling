# Providing Better Compilation Performance Information

When compile times become a problem, how can Scala developers reason about
the relation between their code and compile times?

## Maintenance status

This plugin was created at the [Scala Center](http://scala.epfl.ch) in 2017 and 2018 as the result of the proposal [SCP-10](https://github.com/scalacenter/advisoryboard/blob/main/proposals/010-compiler-profiling.md), submitted by a [corporate member](https://scala.epfl.ch/corporate-membership.html) of the board. The Center is seeking new corporate members to fund activities such as these, to benefit the entire Scala community.

[Version 1.0](https://github.com/scalacenter/scalac-profiling/releases/tag/v1.0.0) of the plugin supports Scala 2.12.

The plugin is now community-maintained, with maintenance overseen by the Center. Thanks to volunteer contributors, there is now a 1.1.0-RC1 [release candidate](https://github.com/scalacenter/scalac-profiling/releases/tag/v1.1.0-RC1) targeting Scala 2.13 (in addition to 2.12). We invite interested users to test the release candidate and submit further improvements.

## Install

Add `scalac-profiling` in any sbt project by specifying the following project
setting.

```scala
addCompilerPlugin("ch.epfl.scala" %% "scalac-profiling" % "<version>" cross CrossVersion.full)
```

## How to use

To learn how to use the plugin, read [Speeding Up Compilation Time with `scalac-profiling`](https://www.scala-lang.org/blog/2018/06/04/scalac-profiling.html) in the scala-lang blog.

### Compiler plugin options

All the compiler plugin options are **prepended by `-P:scalac-profiling:`**.

* `show-profiles`: Show implicit searches and macro expansions by type and
  call-site.
* `sourceroot`: Tell the plugin what is the source directory of the project.
  Example: `-P:scalac-profiling:sourceroot:$PROJECT_BASE_DIR`.
* `print-search-result`: Print the result retrieved by an implicit search.
  Example: `-P:scalac-profiling:print-search-result:$MACRO_ID`.
* `generate-macro-flamegraph`: Generate a flamegraph for macro expansions. The
  flamegraph for implicit searches is enabled by default.
* `print-failed-implicit-macro-candidates`: Print trees of all failed implicit
  searches that triggered a macro expansion.
* `no-profiledb`: Recommended. Don't generate profiledb (will be on by default
  in a future release).
* `show-concrete-implicit-tparams`: Use more concrete type parameters in the
  implicit search flamegraph. Note that it may change the shape of the
  flamegraph.

## Goal of the project

The goal of this proposal is to allow Scala developers to optimize their
codebase to reduce compile times, spotting inefficient implicit searches,
expanded macro code, and other reasons that slow down compile times and
decrease developer productivity.

This repository holds the compiler plugin and a fork of mainstream scalac
that will be eventually be merged upstream. This work is prompted by [Morgan
Stanley's proposal](PROPOSAL.md) and was approved in our last advisory board.

## Scalac status

The required changes to the compiler, [Scalac](http://github.com/scala/scala), are
the following:

1. [Collect all statistics and optimize checks](https://github.com/scala/scala/pull/6034).
1. [Initialize statistics per global](https://github.com/scala/scala/pull/6051).
1. [Add extra timers and counters](https://github.com/scala/scala/pull/6067).

## Information about the setup

The project uses a forked scalac version that is used to compile both the
compiler plugin and several OSS projects from the community. The integration
tests are for now [Circe](https://github.com/circe/circe) and
[Monocle](https://github.com/julien-truffaut/Monocle), and they help us look
into big profiling numbers and detect hot spots and misbehaviours.

If you think a particular codebase is a good candidate to become an integration test, please [open an issue](https://github.com/scalacenter/scalac-profiling/issues/new).

## Plan

The [proposal](PROPOSAL.md) is divided into three main areas:

1. Data generation and capture.
1. Data visualisation and comparison.
1. Reproducibility.

How to tackle each of these problems to make the implementation successful?

### Data generation and capture

The generation of data comes from the guts of the compiler. To optimize for
impact, the collection of information is done in two different places (a
compiler plugin and a forked scalac).

#### Project structure

1. [A forked scalac](scalac/) with patches to collect profiling information.
   <s>All changes are expected to be ported upstream.</s> This fork is not required
   anymore because all the changes are already present in Scala 2.12.5.
1. [A compiler plugin](plugin/) to get information from the macro infrastructure independently
   of the used Scalac version.
1. [Profiledb readers and writers](profiledb/) to allow IDEs and editors to read and write profiledb's.
1. [A proof-of-concept vscode integration](vscode-scala/) that displays the data collected from
   the profiledb.
1. [An sbt plugin for reproducibility](sbt-plugin/) that warms up the compiler before profiling.

The work is split into two parts so that Scala developers that are stuck in previous Scala
versions can use the compiler plugin to get some profiling information about macros.

This structure is more practical because it allow us to evolve things faster in the compiler
plugin, or put there things that cannot be merged upstream.

### Data visualisation and comparison

The profiling data will be accessible in two different ways (provided that
the pertinent profiling flags are enabled):

1. A summary of the stats will be printed out in every compile run.
1. A protobuf file will be generated at the root of the class files directory.
   * The file is generated via protobuf so that it's backwards and forwards binary compatible
   * The protobuf file will contain all the profiling information.

Why a protobuf file instead of a JSON file? Forwards and backwards binary
compatibility is important -- we want our tooling to be able to read files
generated by previous or upcoming versions of the compiler. Our goal is to
create a single tool that all IDEs and third-party tools use to parse and
interpret the statistics from JARs and compile runs.

We're collaborating with [Intellij](https://github.com/JetBrains/intellij-scala) to provide
some of the statistics within the IDE (e.g. macro invocations or implicit searches per line).
We have some ideas to show this information as [heat map](https://en.wikipedia.org/wiki/Heat_map) in the future.

### Reproducibility

Getting reproducible numbers is important to reason about the code and
identifying when a commit increases or decreases compile times with
certainty.

To do so, several conditions must be met: the compiler must be warmed up, the
load in the running computer must be low, and the hardware must be tweaked to
disable options that make executions non reproducible (like Turbo Boost).

However, this warming up cannot be done in an isolated scenario as [Scalac's
benchmarking](https://github.com/scala/compiler-benchmark) infrastructure
does because it doesn't measure the overhead of the build tool calling the
compiler, which can be significant (e.g. in sbt).

As a result, reproducibility must be achieved in the build tool itself. The goal
of this project is to provide an sbt plugin that warms up the compiler by a configurable
amount of time. It also bundles recommendations and tips on how and where to run compilation.

## Collected data

In the following sections, I elaborate on the collected data that we want to
extract from the compiler as well as technical details for every section in
the [original proposal](PROPOSAL.md).

### Information about macros

Per call-site, file and total:

- [x] How many macros are expanded?
- [x] How long do they take to run?
- [x] How many tree nodes do macros create?

### Information about implicit search

Getting hold of this information requires changes in mainstream scalac.

Per call-site, file and total:

- [x] How many implicit searches are triggered per position?
- [x] How many implicit searches are triggered for a given type?
- [x] How long implicit searches take to run?
- [x] How many implicit search failures are?
- [x] How many implicit search hits are?
- [x] What's the ratio of search failures/hits?

### Results

These are the requirements that the proposal lays out.

Note that in some cases, this plugin provides more information than the requested by the
original proposal.

#### What the proposal wants

- [x] Compilation time totally (*this is provided by `-Ystatistics`*)
- [x] Macro details
  - [x] Time per file
  - [x] Time per macro
    - [x] Invocations
    - [x] Per type
    - [x] Total time
  - [x] Flamegraph of all macros
- [x] Implicit search details (time and number)
  - [x] By type
  - [x] By invocation (only number for now)
  - [x] By file (can be aggregated from the "by invocation" data)
  - [x] Flamegraph of all the implicit searches
- [x] User time, kernel time, wall clock, I/O time.<br>
      This feature was **already provided by Scalac**, implemented in [this PR](https://github.com/scala/scala/pull/5848).
- [x] Time for flagged features (for certain features – e.g. optimisation)
  - The best way to capture this information is running statistics for the compiler with
  and without optimization, and compare the profiles. There are also some extra counters.
- [x] Time resolving types from classpath
  - [x] Total
