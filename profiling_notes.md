# Profiling OCaml code

There are many ways of profiling OCaml code. Here we describe several we have used that should get you up and going.


## OCaml gprof support

TODO
 - building to get all the stdlib covered
 - building to get opam covered

## Callgrind

A good introduction to callgrind, if you've never used it, is here:
 https://web.stanford.edu/class/archive/cs/cs107/cs107.1196/resources/callgrind

You can run callgrind from a standard valgrind install on your OCaml binary (as with any other binary):
```
valgrind --tool=callgrind program-to-run program-argements
callgrind_annotate callgrind.out.pid
kcachegrind callgrind.out.pid
```

After a while you will start to notice that you have strange symbols that don't make sense or that you can't see. There are a couple of things you can fix:
 - You need to make sure you have the sources available for all dependent packages (incl the compiler). This can be done in opam with:
 ```
opam reinstall switch --keep-build-dir
 ```
 - You will want to have the debugging symbols available for the standard Linux libraries; refer to your distribution for how to do that. 

With that done, you may still have some odd functions appearing. This can be because the OCaml compilers (at least before 4.08) don't export the size several assembler functions (particularly `caml_c_call`) into the ELF binary. Ideally the OCaml runtime would export the size of in the ELF information, but right now it does not have `.size` directives in the asembler. 

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
valgrind --tool=callgrind --fn-skip=_init --fn-skip=caml_c_call -- program-to-run program-argements
```


## perf record profiling

perf Examples:
 http://www.brendangregg.com/perf.html

TODO

## strace

TODO

## OCaml Landmarks

TODO

## Compiler explorer

TODO

https://godbolt.org

