## Benchmarking the OCaml compiler: what we have learnt and suggestions for the future

_Tom Kelly ([ctk21@cl.cam.ac.uk](mailto:ctk21@cl.cam.ac.uk))_
_Stephen Dolan_
_Sadiq Jaffer_
_Anil Madhavapeddy_

Our goal is to provide statistically meaningful benchmarking of the OCaml compiler
to aid our multicore upstreaming efforts through 2019. In particular we wanted an
infrastructure that operated continuously on all compiler commits and could handle
multiple compiler variants (e.g. flambda). This allows developers to be sure that no
performance regressions slip through and also guides efforts as to where to focus when
introducing new features like multicore which can have a non-trivial impact on
performance, as well as introduce additional non-determinism into measurements due
to parallelism.

This document describes what we did to build continuous benchmarking
websites[^1] that take a controlled experiment approach to running the
operf-micro and sandmark benchmarking suites against tracked git branches of
the OCaml compiler. 

Our high level aims were:

*   Try to reuse OCaml benchmarks that exist and cover a range of micro and macro use cases. 
*   Try to incorporate best-practice and reuse tools where appropriate by looking at what other compiler development communities are doing. 
*   Provide visualizations that are easy to use for OCaml developers making it easy and less time consuming to merge complex features like multicore without performance regressions.  
*   Document the steps to setup the system including its experimental environment for benchmarking so that the knowledge is captured for others to reuse.

This document is structured as follows: we survey the OCaml benchmark packages
available; we survey the infrastructure other communities are using to catch
performance regressions in their compilers; we highlight existing continuous
benchmarks of the OCaml compiler; we describe the system we have built from
selecting the code, running in a controlled environment through to how it is
displayed; we summarise what we have learnt; finally we discuss directions
future work that could follow. 

### Survey of OCaml benchmarks

#### Operf-micro[^2]

Operf-micro collates a collection of micro benchmarks originally put together
to help with the development of flambda. This tool compiles a micro-benchmark
function inside a harness. The harness runs the tested function in a loop to
generate samples x(n) where n is the number of loop iterations. The harness
will collate data for multiple iteration lengths n which provides coverage of a
range of garbage collector behaviour. The number of samples _(n, x(n))_ is
determined by a runtime quota target. Once this _(n, x(n))_ data is collected, it
is post processed using regression to provide the execution time of a single
iteration of the function.

The tool is designed to have minimal external dependencies and can be run
against a test compiler binary without additional packages. The experimental
method of running for multiple embedded iterations is also used by Jane
Street’s `core_bench`[^3] and Haskell’s criterion[^4].

#### operf-macro[^5] and sandmark[^6]

Operf-macro provides a framework to define and run macro-benchmarks. The
benchmarks themselves are OPAM packages and the compiler versions are OPAM
switches. The use of OPAM is to handle the dependencies of larger benchmarks
and to handle the maintenances of the macro-benchmark code base. Sandmark takes
a similar approach to operf-macro but utilizes OPAM v2 and pins the opam repo
to be internally defined so that all the code is fixed from within the one
project. OCaml Labs has been adding new benchmarks to sandmark in order to
increase coverage for multicore testing. 

For the purposes of our benchmarking we decided to run both the operf-micro and
sandmark packages.

### How other compiler communities handle continuous benchmarking

#### Python (CPython and PyPy)

The Python community has continuous benchmarking both for their CPython[^7] and
PyPy[^8] runtimes. The benchmarking data is collected by running the Python
Performance Benchmark Suite[^9]. The open-source web application Codespeed[^10]
provides a front end to navigate and visualize the results. Codespeed is
written in Python on Django and provides views into the results via a revision
table, a timeline by benchmark and comparison over all benchmarks between
tagged versions. Codespeed has been picked up by other projects as a way to
quickly provide visualizations of performance data across code revisions.

#### LLVM

LLVM has a collection of micro-benchmarks in their C/C++ compiler test suite.
These micro-benchmarks are built on the google-benchmark library and produce
statistics that can be easily fed downstream. They also support external tests
(for example SPEC CPU 2006). LLVM have performance tracking software called LNT
which drives a continuous monitoring site[^11]. While the LNT software is
packaged for people to use in other projects, we could not find another project
using it for visualizing performance data and at first glance did not look easy
to reuse. 

#### GHC

The Haskell community have performance regression tests that have hardcoded
values which trip a continuous-integration failure. This method has proved
painful for them[^12] and they have been looking to change it to a more data
driven approach[^13]. At this time they did not seem to have infrastructure
running to help them. 

#### Rust

Rust have built their own tools to collect benchmark data and present it in a
web app[^14]. This tool has some interesting features:

*   It measures both compile time and runtime performance.
*   They are committing the data from their experiments into a github repo to
    make the data available to others. 

### Relation to existing OCaml compiler benchmarking efforts

#### OCamlPro Flambda benchmarking

OCamlPro put together the operf-micro, operf-macro and
[http://bench.flambda.ocamlpro.com/](http://bench.flambda.ocamlpro.com/) site
to provide a benchmarking environment for flambda. Our work builds on these
tools and is inspired by them. 

#### Initial OCaml multicore benchmarking site

OCaml Labs put together an initial multicore benchmarking site
[http://ocamllabs.io/multicore](http://ocamllabs.io/multicore). This built on
the OCamlPro flambda site by (i) implementing visualization in a single
javascript library; (ii) upgrading to OPAM v2; (iii) updating the
macro-benchmarks to more recent version.[^15] The work presented here
incorporates the experience of building that site and builds on it.

### What we put together

We decided to use operf-micro and sandmark as our benchmarks. We went with
Codespeed as a visualization tool since it looked easy to setup and had
instances running in multiple projects. We also wanted to try an open source
tool where there is already a community in place. 

The system runs a pipeline: it determines that a commit is of interest on a
branch timeline it is tracking. It builds the compiler for that commit given
the hash. It then runs either operf-micro or sandmark on a clean experimental
CPU. The data is uploaded into Codespeed to allow users to view and explore the
data. 

#### Transforming git commits to a timeline

Mapping a git branch to a linear timeline requires a bit of care. We use the
following mapping, we take a branch tag and ask for commits using the
first-parent option[^16]. We then use the commit date from this as the
timestamp for a commit. In most repositories this will give a linear code order
that makes sense, even though the development process has been decentralised.
It works well for Github style pull-requests development.

#### Experimental setup

We wanted to try to remove sources of noise in the performance measurements
where possible. We configured our x86 Linux machines as follows:[^17]

*   **Hyperthreading**: Hyperthreading was configured off in the machine BIOS
    to avoid cross-talk and resource sharing for a CPU.
*   **Turbo boost**: Turbo boost was disabled on all CPUs to ensure that the
    processor did not enter turbo which could be throttled by external factors.
*   **Pstate configuration:** We set the power state explicitly to performance
    rather than powersave. 
*   **Linux CPU isolation**: We isolated several cores on our machine using the
    Linux isolcpus kernel parameter. This allowed us to be sure that only the
    benchmarking process was scheduled on a core and that core did not schedule
    other processes on the system.
*   **Interrupts: **We shifted all operating system interrupt handling away
    from the isolated CPUs we used for benchmarking.
*   **ASLR (address space layout randomisation): **We switched this off on our
    benchmarking runs to make experiments repeatable.

With this configuration, we found that we could run benchmarks such that they
were very likely to give the same result when rerun at another time. 

#### Visualization with Codespeed

Once the experimental results have been collected, we upload them into the
codespeed visualization tool which presents a web interface for browsing the
results. This tool presents three ways to interact with the data:

*   **Changes**: This view allows you to browse the raw results across all
    benchmarks collected for a particular commit, build variant and machine
    environment.
*   **Timeline: **This view allows you to look at a given benchmark through
    time on a given machine environment. You can compare multiple build
    variants on the same graph. It also has a grid view that gives a panel
    display of timelines for all benchmarks. It can be a good way to spot
    regressions in the performance and see if the regression is persistent in
    nature.
*   **Comparison: **This view allows you to compare tagged versions of the code
    to each other (including the latest commit of a branch) across all
    benchmarks. This can be a good compact view to ask questions like: “which
    benchmarks is 4.08 faster or slower than 4.07.1?”

The tool provides cross-linking between commits and Github which makes it quick
to drill into the specific code diff of a commit.

### What we learnt

#### Very useful tool to discover performance regressions with multicore

The benchmarking over many compiler commits and visualization tool has been
very useful in enabling us to determine which test cases have performance
regressions with multicore. Because the experimental environment is clean, it
makes the results we see much more repeatable when we investigate things that
look odd. The timeline view is also able to give clues as to which area of the
code changed to cause a performance regression. When performance regressions
are fixed, the benchmarks are automatically run and so provides a ratchet to
know you are making things better without additional manual overhead. 

#### Getting a clean experimental setup is possible but takes some care

We found that getting our machine environment into a state where we could rerun
benchmarks and get repeatable performance numbers was possible, but takes some
care. Often we only discovered we had a problem by looking at large numbers of
benchmarks and diving into commits that caused performance swings we didn’t
understand (for example changes to documentation that altered performance).
Hopefully by collecting the configuration information together in a single
place other people can quickly setup clean environments for their own
benchmarking. Having a clean environment allowed us to really probe the
microstructure effects with x86 performance counters in a rigorous and
iterative way. 

#### Address space layout randomization (ASLR)

This was found following investigation of some benchmark instability coming
from timeline changes but with commits not touching compiler code. We tracked
down the ~10-15% swings were being caused by branch miss-prediction in some
operf-micro format benchmarks that were sensitive to how the address space was
laid out with randomization. We switched off ASLR so that our benchmarks would
be repeatable and an identical binary between commits would be very likely to
give the same result on a single run. User binaries in the wild will likely run
with ASLR on, but we have configured our environment to make benchmarks
repeatable without the need to take the average of many sample. It remains an
open area to benchmark in reasonable time and allow easy developer interaction
while investigating performance under the presence of ASLR[^18]. 

#### Code layout can matter in surprising ways

It is well known that code layout can matter when evaluating the performance of
a binary on a modern processor. Modern x86 processors are often deeply
pipelined and can issue multiple decoded uops per cycle. In order to maintain
good performance it is critical that there is a ready stream of decoded uops
available to be executed on the back end of the processor. There are many ways
a uop could be unavailable.

A well known way for a uop to be unavailable is because the processor is
waiting for it to be fetched from the memory hierarchy. Within the operf-micro
benchmarking suite on the hardware we used, we didn’t see any performance
instability coming from CPU interactions with the instruction or data cache or
main memory. The instruction path memory bottleneck may still be worth
optimizing for and we expect that changes which do optimize this area will be
seen in the sandmark macro-benchmarks[^19]. There are a collection of
approaches being used to improve code layout for instruction memory latency
using profile layout[^20].

We were surprised by the size of the performance impact that code layout can
have due to how instructions pass through the front end of an x86 processor on
their way to being issued[^21]. On x86 alignment of blocks of instructions and
how decoded uops are packed into decode stream buffers (DSB)  can impact the
performance of smaller hot loops in code. Examples of some of the mechanical
effects are:

*   DSB Throughput and alignment
*   DSB Thrashing and alignment
*   Branch-predictor effects and alignment

At a high-level this can be thought of as taking blocks of x86 code and mapping
them into a scarce uops cache. If the blocks of x86 code in the hot path are
unfortunately aligned or have high jump and/or branch complexity then this can
make the decoder and uops cache a bottleneck in your code. If you want to learn
more about the causes of performance swings due to hot code placement in IA
there is a good LLVM dev talk given by Zia Ansari (Intel)
[https://www.youtube.com/watch?v=IX16gcX4vDQ](https://www.youtube.com/watch?v=IX16gcX4vDQ). 

We came across some of these hot-code placement effects in our OCaml
micro-benchmarks when diving into performance swings that we didn’t understand
by looking at the commit diff. We have an example that can be downloaded[^22]
which alters the alignment of a hot loop in an OCaml microbenchmark (without
changing the individual instructions in the hot loop) that lead to a ~15% range
in observed performance on our Xeon E5-2430L machine. From a compiler writer’s
perspective this can be a difficult area, we don’t have any simple solutions
and other compiler communities also struggle with how to tame these
effects[^23].

It is important to realise that a code layout change can lead to changes in the
performance of hot loops. The best way to proceed when you see an unexpected
performance swing on x86 is to look at performance counters around the DSB and
branch predictor. We hope to add selected performance counter data to the
benchmarks we publish in the future. 

#### Logistical issues when scaling up

To run our experiments in a controlled fashion, we need to schedule
benchmarking runs so that they do not get impacted by external factors. Right
now we have been managing with some scripts and manual placement, but we are
quickly approaching the point where we need to have some distributed job
scheduling and management to assist us. This should allow us to handle multiple
architectures, operating systems and benchmarking of more variants of the
compiler. We have yet to decide what software to use for task scheduling and
management. 

### Next Steps (as of April 2019)

#### Managing a heterogeneous cluster of machines, operating systems and benchmarks in a low-overhead way

We want to support a large number of machine architectures, operating systems
and benchmarking suites. To do this we need to control a very large number of
concurrent benchmarking experiments where process scheduling is tightly
controlled and error management structured. A core component needed will be a
distributed job scheduler. On top of the scheduler, we need a system to handle
iterative result generation, backfilling results for new benchmarks and
regeneration of additional data points on demand. By using a scheduler it
should be possible to get a better resource utilization than under manual block
scheduling.   

#### Publishing data for downstream tools

We would like to provide a mechanism for publishing the raw data we collect
from running benchmarking experiments. We can see two possible ideas here: i)
similar to Rust we could commit code to a github hosted repo and then the data
can be served using the git toolchain; ii) we could provide a GraphQL style
server interface which would also be a component of front-end web app
visualization tools. This data format could be closely coupled with what
benchmarking suites publish as the result of running benchmark code. 

#### Modernise the front-end visualization and incorporate multi-dimensional data navigation

We have successfully used Codespeed to visualize our data. This codebase is
based on old Javascript libraries and has internal mechanisms for transporting
data. It is not a trivial application to extend or adapt. A new front-end based
on modern web development technologies which also handles multi-dimensional
benchmark output easily would be useful. 

#### Ability to sandbox the benchmarks while containing noise for pull-request testing

It would be useful for developer workflow to have the ability to benchmark
pull-request branches. There are challenges to doing this while keeping the
experiments controlled. One approach would be to setup a sandboxed environment
that is as close to the bare metal as possible but maintains the low noise
environment we have from using low-level Linux options. Resource usage control
and general scheduling would also need to be considered in such a multi-user
system. 

#### Publish performance counter and GC data coming from benchmarks

We would like to publish multi-dimensional data from the benchmarks and have a
structured way to interpret them. There is a bit of work to standardise the
data we get out of the benchmarks suites and integrating the collection of
performance counters would also need doing. Once the data is collected, there
would be a little work to present visualizations of the data.  

#### Experimenting with x86 code alignment emitted from the compiler

LLVM and GCC have a collection of options available to alter the x86 code
alignment. Some of these get enabled at higher optimization levels and other
options help with debugging.[^24] For example, it might be useful to be able to
align all jump targets; code size would increase and performance decrease, but
it might allow you to isolate a performance swing to being due to
microstructure alignment when doing a deep dive. 

#### Compile time performance benchmarking

We would like to produce numbers on the compile-time performance of compiler
versions. The first step would be to decide on a representative corpus of OCaml
code to benchmark (could be sandmark).  

#### How to tell if your benchmark suite is giving good coverage of the compiler optimizations

It would be useful to have some coverage metrics of a benchmarking suite
relative to the compiler functionality. This might reveal holes in the
benchmarking suite and having an understanding of which bits of the
optimization functionality are engaged on a given benchmark could be revealing
in itself. 

#### Using the infrastructure for continuous benchmarking of general OCaml binaries

It would be interesting to try our infrastructure on a larger OCaml project
where tracking its performance is important to the community of developers and
users for it. The compiler version would be fixed but the project code would
move through time. One idea we had is odoc but others projects might be more
interesting to explore. 

### Conclusions

We are satisfied that we have produced infrastructure that will prove effective
for continuous performance regression of multicore. We are hoping that the
infrastructure can be more generally useful in the wider community. We hope
that by sharing our experiences with benchmarking OCaml code we can help people
quickly realise good benchmarking experiments and avoid non-trivial pitfalls. 

<!-- Footnotes themselves at the bottom. -->
## Notes

[^1]:
     [http://bench.ocamllabs.io](http://bench.ocamllabs.io) which presents operf-micro benchmark experiments and [http://bench2.ocamllabs.io](http://bench2.ocamllabs.io) which presents sandmark based benchmark experiments. 

[^2]:
     [https://github.com/OCamlPro/operf-micro](https://github.com/OCamlPro/operf-micro) - tool code
    [https://hal.inria.fr/hal-01245844/document](https://hal.inria.fr/hal-01245844/document) - “Operf: Benchmarking the OCaml Compiler”, Chambart et al

[^3]:
     Code for core_bench [https://github.com/janestreet/core_bench](https://github.com/janestreet/core_bench) and blog post describing core_bench [https://blog.janestreet.com/core_bench-micro-benchmarking-for-ocaml/](https://blog.janestreet.com/core_bench-micro-benchmarking-for-ocaml/)

[^4]:
     Haskell code for criterion [https://github.com/bos/criterion](https://github.com/bos/criterion) and tutorial on using criterion [http://www.serpentine.com/criterion/tutorial.html](http://www.serpentine.com/criterion/tutorial.html) 

[^5]:
     Code and documentation for operf-macro [https://github.com/OCamlPro/operf-macro](https://github.com/OCamlPro/operf-macro) 

[^6]:
     Code for sandmark [https://github.com/ocamllabs/sandmark](https://github.com/ocamllabs/sandmark) 

[^7]:
     CPython continuous benchmarking site [https://speed.python.org/](https://speed.python.org/) 

[^8]:
     PyPy continuous benchmarking site [http://speed.pypy.org/](http://speed.pypy.org/) 

[^9]:
      Python Performance Benchmark Suite [https://pyperformance.readthedocs.io/](https://pyperformance.readthedocs.io/) 

[^10]:
     Codespeed web app for visualization of performance data [https://github.com/tobami/codespeed](https://github.com/tobami/codespeed) 

[^11]:
     LNT software [http://llvm.org/docs/lnt/](http://llvm.org/docs/lnt/) and the performance site [https://lnt.llvm.org/](https://lnt.llvm.org/) 

[^12]:
     Description of Haskell issues with hard performance continous integration tests [https://ghc.haskell.org/trac/ghc/wiki/Performance/Tests](https://ghc.haskell.org/trac/ghc/wiki/Performance/Tests) 

[^13]:
     ‘2017 summer of code’ work on improving Haskell performance integration tests [https://github.com/jared-w/HSOC2017/blob/master/Proposal.pdf](https://github.com/jared-w/HSOC2017/blob/master/Proposal.pdf)  

[^14]:
     Live tracking site [https://perf.rust-lang.org/](https://perf.rust-lang.org/) and code for it [https://github.com/rust-lang-nursery/rustc-perf](https://github.com/rust-lang-nursery/rustc-perf)  

[^15]:
     More information here: [http://kcsrk.info/multicore/ocaml/benchmarks/2018/09/13/1543-multicore-ci/](http://kcsrk.info/multicore/ocaml/benchmarks/2018/09/13/1543-multicore-ci/) 

[^16]:
     A nice description of why first-parent helps is here [http://www.davidchudzicki.com/posts/first-parent/](http://www.davidchudzicki.com/posts/first-parent/) 

[^17]:
     For more details please see [https://github.com/ocaml-bench/ocaml_bench_scripts/#notes-on-hardware-and-os-settings-for-linux-benchmarking](https://github.com/ocaml-bench/ocaml_bench_scripts/#notes-on-hardware-and-os-settings-for-linux-benchmarking)

[^18]:
     There is some academic literature in this direction; for example “Rigourous benchmarking in reasonable time”, Kalibera et al [https://dl.acm.org/citation.cfm?id=2464160](https://dl.acm.org/citation.cfm?id=2464160) and “STABILIZER: statistically sound performance evaluation”, Curtsinger et al,  [https://dl.acm.org/citation.cfm?id=2451141](https://dl.acm.org/citation.cfm?id=2451141). However randomization approaches need to take care that the layout randomization distribution captures the relevant features of layout randomization that real user binaries will see from ASLR, link-order or dynamic library linking.

[^19]:
     We feel that there must be some memory latency bottlenecks in the sandmark macro benchmarking suite, but we have yet to deep-dive and investigate a performance instability due to noise in memory latency for fetching instructions to execute. That is the performance may be bottlenecked on instruction memory fetch, but we haven’t seen instruction memory fetch latency being very noisy between benchmark runs in our setup.

[^20]:
     Google’s AutoFDO [https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45290.pdf](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45290.pdf), Facebook’s HFSort [https://research.fb.com/wp-content/uploads/2017/01/cgo2017-hfsort-final1.pdf](https://research.fb.com/wp-content/uploads/2017/01/cgo2017-hfsort-final1.pdf) and Facebook’s Bolt [https://github.com/facebookincubator/BOLT](https://github.com/facebookincubator/BOLT) are recent examples to report imporvements in some deployed data-centre workloads. 

[^21]:
     The effects are not limited to x86 as it has also been observed on ARM Cortex-A53 and Cortex-A57 processors with LLVM [https://www.youtube.com/watch?v=COmfRpnujF8](https://www.youtube.com/watch?v=COmfRpnujF8)   

[^22]:
     The OCaml example is here [https://github.com/ocaml-bench/ocaml_bench_scripts/tree/master/stability_example](https://github.com/ocaml-bench/ocaml_bench_scripts/tree/master/stability_example), for the super curious there are yet more C++ examples to be had (although not necessarily the same underlying micro mechanism and all dependent on processor) [https://dendibakh.github.io/blog/2018/01/18/Code_alignment_issues](https://dendibakh.github.io/blog/2018/01/18/Code_alignment_issues)  

[^23]:
     The Q&A at LLVM dev meeting videos [https://www.youtube.com/watch?v=IX16gcX4vDQ](https://www.youtube.com/watch?v=IX16gcX4vDQ) and [https://www.youtube.com/watch?v=COmfRpnujF8](https://www.youtube.com/watch?v=COmfRpnujF8) are examples of discussion around the area. The general area of code layout is also a problem area in the academic community “Producing wrong data without doing anything obviously wrong!”, Mytkowicz et al [https://dl.acm.org/citation.cfm?id=1508275](https://dl.acm.org/citation.cfm?id=1508275) 

[^24]:
     For more on LLVM alignment options, see [https://dendibakh.github.io/blog/2018/01/25/Code_alignment_options_in_llvm](https://dendibakh.github.io/blog/2018/01/25/Code_alignment_options_in_llvm)
