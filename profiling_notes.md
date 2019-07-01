# Profiling OCaml code

There are many ways of profiling OCaml code. Here we describe several we have used that should get you up and going.


## perf record profiling

Basic setup
```
perf record --call-graph dwarf -- program-to-run program-arguments
perf report -G
perf report --call-graph
perf report --children
perf report --no-children
perf report --no-children --kallsyms /proc/kallsyms
```

echo 0 | sudo tee /proc/sys/kernel/kptr_restrict

Installing debug symbols on ubuntu
 https://wiki.ubuntu.com/Debug%20Symbol%20Packages

In particular the kernel symbols:
 `apt-get install linux-image-$(uname -r)-dbgsym`
and libgmp
 `apt-get install `

Pointer to perf Examples:
 http://www.brendangregg.com/perf.html

Limitations:
 - sometimes can get strange annotations
 - it's statistical so you don't get full coverage or call counts
 - the OCaml runtime functions can be confusing if you aren't familiar with them

Pros:
 - it's fast as it samples the running program
 - the cli can be a very quick way of getting a callchain to start from with a big program

## OCaml gprof support

Documentation on gprof is here:
 https://sourceware.org/binutils/docs/gprof/

To run your OCaml program with gprof you need to do:
```
ocamlopt -o myprog -p other-options files
./myprog
gprof myprog
```

The call count is accurate and the timing information is statistical but your program now has extra code which wouldn't be in a release binary.

Pros:
 - fairly quick way to get a call graph without having to use other tools
 - call counts are accurate

Limitations:
 - your program is instrumented so isn't what you will really run in production
 - not obvious how to profile 3rd party packages built through OPAM

TODO:
 - is there a way to build to get all the stdlib covered?
 - is there a way to build to get opam covered?

## Callgrind

A good introduction to callgrind, if you've never used it, is here:
 https://web.stanford.edu/class/archive/cs/cs107/cs107.1196/resources/callgrind

You can run callgrind from a standard valgrind install on your OCaml binary (as with any other binary):
```
valgrind --tool=callgrind program-to-run program-arguments
callgrind_annotate callgrind.out.pid
kcachegrind callgrind.out.pid
```

After a while you will start to notice that you have strange symbols that don't make sense or that you can't see. There are a couple of things you can fix:
 - You need to make sure you have the sources available for all dependent packages (incl the compiler). This can be done in opam with:
 ```
opam reinstall switch --keep-build-dir
 ```
 - You will want to have the debugging symbols available for the standard Linux libraries; refer to your distribution for how to do that.

With that done, you may still have some odd functions appearing. This can be because the OCaml compilers (at least before 4.08) don't export the size several assembler functions (particularly `caml_c_call`) into the ELF binary. Ideally the OCaml runtime would export the size of in the ELF information, but right now it does not have `.size` directives in the assembler.

To fix this, you will need to build a patched valgrind which will let you see them.

The patch against 3.15.0 (available here http://www.valgrind.org/downloads/repository.html) is this:
```
 diff --git a/coregrind/m_debuginfo/readelf.c b/coregrind/m_debuginfo/readelf.c
index b982a838a..8b75b260b 100644
--- a/coregrind/m_debuginfo/readelf.c
+++ b/coregrind/m_debuginfo/readelf.c
@@ -253,6 +253,7 @@ void show_raw_elf_symbol ( DiImage* strtab_img,
    to piece together the real size, address, name of the symbol from
    multiple calls to this function.  Ugly and confusing.
 */
+#define ALLOW_ZERO_ELF_SYM 1
 static
 Bool get_elf_symbol_info (
         /* INPUTS */
@@ -282,7 +283,8 @@ Bool get_elf_symbol_info (
    Bool in_text, in_data, in_sdata, in_rodata, in_bss, in_sbss;
    Addr text_svma, data_svma, sdata_svma, rodata_svma, bss_svma, sbss_svma;
    PtrdiffT text_bias, data_bias, sdata_bias, rodata_bias, bss_bias, sbss_bias;
-#     if defined(VGPV_arm_linux_android) \
+#     if defined(ALLOW_ZERO_ELF_SYM) \
+         ||defined(VGPV_arm_linux_android) \
          || defined(VGPV_x86_linux_android) \
          || defined(VGPV_mips32_linux_android) \
          || defined(VGPV_arm64_linux_android)
@@ -475,7 +477,8 @@ Bool get_elf_symbol_info (
       in /system/bin/linker:  __dl_strcmp __dl_strlen
    */
    if (*sym_size_out == 0) {
-#     if defined(VGPV_arm_linux_android) \
+#     if defined(ALLOW_ZERO_ELF_SYM) \
+         || defined(VGPV_arm_linux_android) \
          || defined(VGPV_x86_linux_android) \
          || defined(VGPV_mips32_linux_android) \
          || defined(VGPV_arm64_linux_android)
```

You can then run the following, which will allow callgrind to skip through `caml_c_call`, to create a potentially more useful callgraph:
```
valgrind --tool=callgrind --fn-skip=_init --fn-skip=caml_c_call -- program-to-run program-arguments
```

Limitations:
 - quite slow to run
 - not supported out of the box; right now to get the function skip you will need to compile your own valgrind
 - recursive functions give confusing total function cost scores which should be ignored
 - the OCaml runtime functions can be confusing if you aren't familiar with them

Pros:
 - the function call counts are accurate
 - the instruction count information is very accurate
 - you can instrument all the way to basic block instructions and jumps taken
 - kcachegrind gives you a graphical front end which can sometimes be easier to navigate
 - with the fn-skip options you can look through the caml_c_call to get a callchain from ocaml through underlying libraries (including the OCaml runtime)


## OCaml Landmarks

Landmarks allow you to instrument at the OCaml level and gives visibility into call chains, call counts and memory allocations in blocks:
https://github.com/LexiFi/landmarks

There are a couple of ways to instrument a program; manual, using manual ppx extensions and fully automatic using a ppx mode.

### Manual instrumentation
You setup your landmarks with `register` and need to call `enter` and `exit` on each landmark you are interested in. Care needs to be taken to handle exceptions so that the `enter`/`exit` pairs match up.

### Manual ppx extensions
You annotate specific OCaml lines with a pre-processing command; e.g. `let[@landmark] f = ...`. This handles all the pairing for you and can be quite convenient. You need to add the ppx to your build.

### Automatic instrumentation
You run the pre-processor with `--auto` and this will instrument all top level functions within a module. You need to add the ppx to your build.


#### ocamlbuild and ppx tags
To get things working with ocamlbuild and landmarks you can add the following to your `_tag` file:
```
ppx(`ocamlfind query landmarks.ppx`/ppx.exe --as-ppx)
```
or for auto instrumentation:
```
ppx(`ocamlfind query landmarks.ppx`/ppx.exe --as-ppx --auto)
```

Limitations:
 - the timing information includes the overhead of any landmarks within another landmark
 - the allocation information includes the overhead of any landmarks within another landmark
 - the addition of the landmark will change the compiled code and can remove inlining opportunities that would occur in a release binary
 - can slow down your binary in some cases

Pros:
 - it's very quick to setup automatic profiling and then determine which functions are the heavy hitters
 - it's easy to selectively profile bits of OCaml code you are interested in

## OCaml Spacetime

The OCaml Spacetime profiler allows you to see where in your program memory is allocated.

Setup an opam switch with Spacetime and all your opam packages:
```
 $ opam switch export opam_existing_universe
 $ opam switch create 4.07.1+spacetime
 $ opam switch import opam_existing_universe
```

NB: you might have a stack size issue with the universe, you can expand it to 128M in bash with `ulimit -S -s 131072`.

In the 4.07.1+spacetime switch, build your binary. Now run the executable, using the environment variable OCAML_SPACETIME_INTERVAL to turn on profiling and specify how frequently Spacetime should inspect the OCaml heap (in milliseconds):
```
 $ OCAML_SPACETIME_INTERVAL=100 program-to-run program-arguments
```
This will output a file of the form `spacetime-<pid>`.

To view the information in this file, we need to process it with `prof_spacetime`. Right now, this only runs on OCaml 4.06.1 and lower. Install as follows:
```
 $ opam switch 4.06.1
 $ opam install prof_spacetime
```
To post-process the results do
```
 $ prof_spacetime process -e <path to your executable> spacetime-<pid>
```
This will produce a `spacetime-<pid>.p` file.

To serve this and interact through a web-browser do:
```
 $ prof_spacetime serve -p spacetime-<pid>.p
```

To look at the data through a CLI interface do:
```
 $ prof_spacetime view -p spacetime-<pid>.p
```

For more on Spacetime and its output see:
 - Jane Street blog post on Spacetime: https://blog.janestreet.com/a-brief-trip-through-spacetime/
 - OCaml compiler Spacetime documentation: https://caml.inria.fr/pub/docs/manual-ocaml/spacetime.html

## strace

This tool is very useful for knowing what system calls your program is making. You can wrap a run of your binary with:
```
strace -o <file for output> program-to-run program-arguments
```

## VTune

Intel's VTune is now available to download here:
 https://software.intel.com/en-us/vtune/choose-download

TODO: usage

## Compiler explorer

The compiler explorer supports OCaml as a language:
  https://godbolt.org

Interesting things to try are:
```
let cmp a b =
  if a > b then a else b

let cmp_i (a:int) (b:int) =
  if a > b then a else b

let cmp_s (a:String) (b:String) =
  if a > b then a else b

```

Or what happens with a closure:
```
let fn ys z =
   let g = ((+) z) in
   List.map g ys

let fn_i ys (z:int) =
   let g = ((+) z) in
   List.map g ys

```

You can also use the `Analysis` mode to see how specific chunks of assembly will be executed (try something like: `--mcpu=skylake --timeline`).


## Other resources

OCaml.org tutorial on performance and profiling:
 https://ocaml.org/learn/tutorials/performance_and_profiling.html

OCamlverse links for optimizing performance:
 https://ocamlverse.github.io/content/optimizing_performance.html

Real world ocaml and it's backend section has a little:
 https://dev.realworldocaml.org/compiler-backend.html#compiling-fast-native-code


