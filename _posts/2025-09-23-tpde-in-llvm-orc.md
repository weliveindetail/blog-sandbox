---
layout: post
categories: post
author: Stefan Gränitz
date: 2024-08-29 12:00:00 +0200
image: https://weliveindetail.github.io/blog/res/2024-omvll-clang-repl.png
preview: summary_large_image
title: "Using TPDE Codegen in LLVM ORC"
description: ""
source: https://github.com/weliveindetail/blog/main/_posts/2023-08-29-omvll-clang-repl.md
---

<style>
  #banner-image {
    margin-bottom: 50px;
  }
  #large-image {
    max-width: min(100%, 230px);
  }
  .center {
    display: block;
    margin: 0 auto;
  }
</style>

![tpde-banner](https://weliveindetail.github.io/blog/res/2025-tpde-orc.png){: #large-image}{: .center}

[TPDE](https://arxiv.org/abs/2505.22610) is a single-pass compiler backend for LLVM that was [open-sourced earlier this year](https://discourse.llvm.org/t/tpde-llvm-10-20x-faster-llvm-o0-back-end/) by [TUM](https://db.in.tum.de). The [documentation shows](https://docs.tpde.org/tpde-llvm-main.html) how to integrate it in custom builds of Clang and Flang. Supported release versions are [LLVM 19](https://github.com/tpde2/tpde/blob/c857798/llvm.ab51eccf88f5.patch) and [LLVM 20](https://github.com/tpde2/tpde/blob/c857798/llvm.616f2b685b06.patch).

### Integration in LLVM ORC JIT

The primary goal of TPDE is low-latency code generation while maintaining reasonable (-O0) code quality. That makes it a perfect fit for a baseline JIT compiler. LLVM's [On-Request Compilation (ORC)](https://llvm.org/docs/ORCv2.html) libraries provide a set of libraries to write JIT compilers for LLVM IR input. ORC uses LLVM's built-in backends for codegen by default, but it's very easy to use TPDE instead thanks to its various extension points!

Let's say we use the `LLJITBuilder` interface to instantiate an off-the-shelf JIT:

```cpp
ExitOnError ExitOnErr;
auto Builder = LLJITBuilder();
std::unique_ptr<LLJIT> JIT = ExitOnErr(Builder.create());
```

The builder provides a set of extension points to customize the JIT instance that it creates.
We can overwrite the `CreateCompileFunction` member to define the codegen component, which
compiles LLVM IR into machine code:
```cpp
Builder.CreateCompileFunction = [](JITTargetMachineBuilder JTMB)
    -> Expected<std::unique_ptr<IRCompileLayer::IRCompiler>> {
  return std::make_unique<TPDECompiler>(JTMB);
};
```

In order to use TPDE here, we just have to wrap it in a compatible class:
```cpp
class TPDECompiler : public IRCompileLayer::IRCompiler {
public:
  TPDECompiler(JITTargetMachineBuilder JTMB)
      : IRCompiler(irManglingOptionsFromTargetOptions(JTMB.getOptions())) {
    Compiler = tpde_llvm::LLVMCompiler::create(JTMB.getTargetTriple());
    assert(Compiler != nullptr && "Unknown architecture");
  }

  Expected<std::unique_ptr<MemoryBuffer>> operator()(Module &M) override;

private:
  std::unique_ptr<tpde_llvm::LLVMCompiler> Compiler;
  std::vector<std::unique_ptr<std::vector<uint8_t>>> Buffers;
};
```

We instantiate TPDE with a target triple like `x86_64-pc-linux-gnu` in the constructor. It works on ELF-based systems and supports 64-bit Intel and ARM architectures (`x86_64` and `aarch64`). For now let's assume this is all we need. Let's implement the actual invocation:
```cpp
Expected<std::unique_ptr<MemoryBuffer>> TPDECompiler::operator()(Module &M) {
  Buffers.push_back(std::make_unique<std::vector<uint8_t>>());
  std::vector<uint8_t> &B = *Buffers.back();

  if (!Compiler->compile_to_elf(M, B)) {
    std::string Msg;
    raw_string_ostream(Msg) << "TPDE failed to compile: " << M.getName();
    return createStringError(std::move(Msg), inconvertibleErrorCode());
  }

  StringRef BufferRef{reinterpret_cast<char *>(B.data()), B.size()};
  return MemoryBuffer::getMemBuffer(BufferRef, "", false);
}
```

We create a new buffer `B` for the binary code and pass it to TPDE together with the module `M` we want to codegen. If TPDE fails, we bail out with an error. If it works, we wrap the result in a `MemoryBuffer` and return it. (LLVM is quite old by now and still uses `char` pointers for binary buffers. This is unfortunate due to the three-types definition of `char` [in the C Standard](https://www.open-std.org/JTC1/SC22/WG14/www/docs/n1256.pdf), but it's hard to change these days.)

Et voila, for basic integration this is it! No need to patch LLVM, this works with official release versions. I set up a sample project to demo it here: https://github.com/weliveindetail/tpde-orc The code there handles a few more details that we will explore now.

### LLJITBuilder has a catch

The `LLJITBuilder` obviously uses LLVM's standard interfaces [including `TargetRegistry`](https://github.com/llvm/llvm-project/blob/release/20.x/llvm/lib/ExecutionEngine/Orc/JITTargetMachineBuilder.cpp#L42). However, this only works, if the respective LLVM target backend is initialized! We have to call `InitializeNativeTarget()` first and thus we have to ship the LLVM backend, even though we don't need it.

We can only avoid this target initialization requirement, if we set up our ORC JIT manullay. This is what [the `tpde-lli` tool does](https://github.com/tpde2/tpde/blob/master/tpde-llvm/tools/tpde-lli.cpp). Before you dive into it though, wait for the next detail!

### LLVM fallback

One reason why TPDE is so fast and compact is that it [doesn't cover all edge cases](https://docs.tpde.org/tpde-llvm-main.html#autotoc_md91) of the LLVM instruction set. The documentation states this rule of thumb:

> Code generated by Clang (-O0/-O1) will typically compile; -O2 and higher will typically fail due to unsupported vector operations.

If your code does contain things like vector ops or non-trvial float types, then it won't compile on the TPDE fast track. It needs a fallback to LLVM. And since this is very likely, most tools will keep shipping the LLVM backend as well. This is easy to implement with [CompileUtils from ORC](https://github.com/llvm/llvm-project/blob/release/20.x/llvm/include/llvm/ExecutionEngine/Orc/CompileUtils.h#L36):

```diff
@@ -29,7 +29,8 @@ static cl::opt<std::string> EntryPoint("entrypoint",
 class TPDECompiler : public IRCompileLayer::IRCompiler {
 public:
   TPDECompiler(JITTargetMachineBuilder JTMB)
-      : IRCompiler(irManglingOptionsFromTargetOptions(JTMB.getOptions())) {
+      : IRCompiler(irManglingOptionsFromTargetOptions(JTMB.getOptions())),
+        TM(ExitOnErr(JTMB.createTargetMachine())) {
     Compiler = tpde_llvm::LLVMCompiler::create(JTMB.getTargetTriple());
     assert(Compiler != nullptr && "Unknown architecture");
   }
@@ -39,6 +40,7 @@ public:
 private:
   std::unique_ptr<tpde_llvm::LLVMCompiler> Compiler;
   std::vector<std::unique_ptr<std::vector<uint8_t>>> Buffers;
+  std::unique_ptr<TargetMachine> TM;
 };
 
 Expected<std::unique_ptr<MemoryBuffer>> TPDECompiler::operator()(Module &M) {
@@ -46,9 +48,8 @@ Expected<std::unique_ptr<MemoryBuffer>> TPDECompiler::operator()(Module &M) {
   std::vector<uint8_t> &B = *Buffers.back();
 
   if (!Compiler->compile_to_elf(M, B)) {
-    std::string Msg;
-    raw_string_ostream(Msg) << "TPDE failed to compile: " << M.getName();
-    return createStringError(std::move(Msg), inconvertibleErrorCode());
+    errs() << "Falling back to LLVM for module: " << M.getName() << "\n";
+    return SimpleCompiler(*TM)(M);
   }
 
   StringRef BufferRef{reinterpret_cast<char *>(B.data()), B.size()};
```

With the change in place, let's compile compile the following IR file:
```llvm
@const_val = global bfloat 0xR4248

define i32 @main() {
entry:
  %c = load bfloat, ptr @const_val
  %i = fptosi bfloat %c to i32
  ret i32 %i
}
```

Command-line output will be:
```
$ ./tpde-orc 02-bfloat.ll
Loaded module: 02-bfloat.ll
[2025-09-25 12:54:03.076] [error] unsupported type: bfloat
[2025-09-25 12:54:03.076] [error] Failed to compile function main
Falling back to LLVM for module: 02-bfloat.ll
Executing main()
Program returned: 50
```

In the above patch we create a new `SimpleCompiler` instance for each fallback case. This adds some overhead, but it should be fine, it's the slow path anyway. And it has an important side-effect that we explore in the next section: it's thread-safe!

### Concurrent compilation on ORC

ORC JIT supports concurrent compilation of concurrent code! This is neat, but it needs attention in customizations. TPDE's `compile_to_elf()` is not thread-safe: It fails if we call it concurrently from multiple threads, but this is exactly what happens in ORC. We need to fix this, but we don't want to create a new TPDE instance for each compile job. This is the fast path!

