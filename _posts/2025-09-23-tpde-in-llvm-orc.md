---
layout: post
categories: post
author: Stefan Gränitz
date: 2024-08-29 12:00:00 +0200
image: https://weliveindetail.github.io/blog-sandbox/res/2025-tpde-orc.png
preview: summary_large_image
title: "Using TPDE Codegen in LLVM ORC"
description: "TPDE is the perfect fit for a baseline JIT compiler, let's see how to use it in ORC JIT!"
source: https://github.com/weliveindetail/blog/main/_posts/2025-09-23-tpde-in-llvm-orc.md
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

![tpde-banner](https://weliveindetail.github.io/blog-sandbox/res/2025-tpde-orc.png){: #large-image}{: .center}

[TPDE](https://arxiv.org/abs/2505.22610){:target="_blank"} is a single-pass compiler backend for LLVM that was [open-sourced earlier this year](https://discourse.llvm.org/t/tpde-llvm-10-20x-faster-llvm-o0-back-end/){:target="_blank"} by [TUM](https://db.in.tum.de){:target="_blank"}. The [documentation shows](https://docs.tpde.org/tpde-llvm-main.html){:target="_blank"} how to integrate it in custom builds of Clang and Flang. Supported release versions are [LLVM 19](https://github.com/tpde2/tpde/blob/c857798/llvm.ab51eccf88f5.patch){:target="_blank"} and [LLVM 20](https://github.com/tpde2/tpde/blob/c857798/llvm.616f2b685b06.patch){:target="_blank"}.

### Integration in LLVM ORC JIT

The primary goal of TPDE is low-latency code generation while maintaining reasonable (-O0) code quality. That makes it a perfect fit for a baseline JIT compiler. LLVM's [On-Request Compilation (ORC)](https://llvm.org/docs/ORCv2.html){:target="_blank"} provides a set of libraries to write JIT compilers for LLVM IR input. ORC uses LLVM's built-in backends for codegen by default, but it's very easy to use TPDE instead thanks to its various extension points!

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

We create a new buffer `B` for the binary code and pass it to TPDE together with the module `M` we want to codegen. If TPDE fails, we bail out with an error. If it works, we wrap the result in a `MemoryBuffer` and return it. (LLVM is quite old by now and still uses `char` pointers for binary buffers. This is unfortunate due to the three-types definition of `char` [in the C Standard](https://www.open-std.org/JTC1/SC22/WG14/www/docs/n1256.pdf){:target="_blank"}, but it's hard to change these days.)

Et voila, for basic integration this is it! No need to patch LLVM, this works with official release versions. We can compile simple LLVM IR code already:
```llvm
> cat 01-basic.ll 
; ModuleID = 'test.ll'
source_filename = "test.ll"

define i32 @main() {
entry:
  %1 = call i32 @custom_entry()
  %2 = sub i32 %1, 123
  ret i32 %2
}

define i32 @custom_entry() {
entry:
  ret i32 123
}
```

I set up a [sample project with a working demo on GitHub](https://github.com/weliveindetail/tpde-orc){:target="_blank"}. This is the output:
```
> ./tpde-orc 01-basic.ll 
Loaded module: 01-basic.ll
Executing main()
Program returned: 0

> ./tpde-orc 01-basic.ll --entrypoint custom_entry
Loaded module: 01-basic.ll
Executing custom_entry()
Program returned: 123
```

The code there handles a few more details that we will explore below. We can already see a 4x speedup with TPDE compared to built-in LLVM codegen:
```
> ./build/tpde-orc --par 1 tpde-orc/03-csmith-tpde.ll
...
Compile-time was: 2200 ms

> ./build/tpde-orc --par 1 tpde-orc/03-csmith-tpde.ll --llvm
...
Compile-time was: 8820 ms
```

### LLJITBuilder has a catch

The `LLJITBuilder` interface we use above obviously incorporates other LLVM standard interfaces [including `TargetRegistry`](https://github.com/llvm/llvm-project/blob/release/20.x/llvm/lib/ExecutionEngine/Orc/JITTargetMachineBuilder.cpp#L42){:target="_blank"}. This is absolutely reasonable, but it adds an unfortunate dependency for us: It only works, if the built-in LLVM target backend is initialized! We have to call `InitializeNativeTarget()` first and thus we have to ship the LLVM backend, even though we don't need it.

We can only avoid this issue if we set up our ORC JIT manually. If this is what you are looking for, have a look [how the `tpde-lli` tool does it](https://github.com/tpde2/tpde/blob/master/tpde-llvm/tools/tpde-lli.cpp){:target="_blank"}. Before you dive into it though, wait for the next detail!

### LLVM fallback

One reason why TPDE is so fast and compact is that it [doesn't cover all edge cases](https://docs.tpde.org/tpde-llvm-main.html#autotoc_md91){:target="_blank"} of the LLVM instruction set. The documentation states this rule of thumb:

> Code generated by Clang (-O0/-O1) will typically compile; -O2 and higher will typically fail due to unsupported vector operations.

If your code does contain things like vector ops or non-trivial float types, then it won't compile on the TPDE fast track. It needs a fallback to LLVM. And since this is very likely, most tools will keep shipping the LLVM backend as well. The fallback is easy to implement with [CompileUtils from ORC](https://github.com/llvm/llvm-project/blob/release/20.x/llvm/include/llvm/ExecutionEngine/Orc/CompileUtils.h#L36){:target="_blank"}:

```diff
@@ -29,7 +29,8 @@ static cl::opt<std::string> EntryPoint("entrypoint",
 class TPDECompiler : public IRCompileLayer::IRCompiler {
 public:
   TPDECompiler(JITTargetMachineBuilder JTMB)
-      : IRCompiler(irManglingOptionsFromTargetOptions(JTMB.getOptions())) {
+      : IRCompiler(irManglingOptionsFromTargetOptions(JTMB.getOptions())),
+        JTMB(std::move(JTMB)) {
     Compiler = tpde_llvm::LLVMCompiler::create(JTMB.getTargetTriple());
     assert(Compiler != nullptr && "Unknown architecture");
   }
@@ -37,9 +38,9 @@ Expected<std::unique_ptr<MemoryBuffer>> TPDECompiler::operator()(Module &M) {
   std::vector<uint8_t> &B = *Buffers.back();
 
   if (!Compiler->compile_to_elf(M, B)) {
-    std::string Msg;
-    raw_string_ostream(Msg) << "TPDE failed to compile: " << M.getName();
-    return createStringError(std::move(Msg), inconvertibleErrorCode());
+    errs() << "Falling back to LLVM for module: " << M.getName() << "\n";
+    auto TM = ExitOnErr(JTMB.createTargetMachine());
+    return SimpleCompiler(*TM)(M);
   }
 
   StringRef BufferRef{reinterpret_cast<char *>(B.data()), B.size()};
@@ -50,6 +51,7 @@ public:
 private:
   std::unique_ptr<tpde_llvm::LLVMCompiler> Compiler;
   std::vector<std::unique_ptr<std::vector<uint8_t>>> Buffers;
+  JITTargetMachineBuilder JTMB;
 };
 
 int main(int argc, char *argv[]) {
```

With this change in place, let's compile the following IR file:
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
```terminal
> ./tpde-orc 02-bfloat.ll
Loaded module: 02-bfloat.ll
[2025-09-25 12:54:03.076] [error] unsupported type: bfloat
[2025-09-25 12:54:03.076] [error] Failed to compile function main
Falling back to LLVM for module: 02-bfloat.ll
Executing main()
Program returned: 50
```

In the above patch we create a new `SimpleCompiler` instance for each fallback case. This adds some overhead, but it should be fine on the slow path anyway. We expect that the majority of jobs in our workload won't trigger the fallback. (Otherwise the TPDE backend doesn't make much sense at all). This implementation has an important side-effect that we explore in the next section: it's thread-safe!

### Concurrent compilation

ORC JIT has built-in support for concurrent compilation. This is neat, but it requires attention when we customize the JIT. Our JIT has a single `TPDECompiler` instance and TPDE's `compile_to_elf()` is not thread-safe. If we enabled concurrent compilation, it would be called from multiple threads and fail.

How can we fix this? We could create a new `tpde_llvm::LLVMCompiler` instance for each call to `TPDECompiler::operator()`, but it adds an overhead of `O(#jobs)` that we don't like on the fast path. Essentially, we want to avoid calling into `compile_to_elf()` while there is another call in-flight on the same instance. Making the `TPDECompiler` instance thread-local guarantees that and reduces the overhead to `O(#threads)`. It's a very simple change as well:

```diff
@@ -32,7 +32,6 @@ public:
   TPDECompiler(JITTargetMachineBuilder JTMB)
       : IRCompiler(irManglingOptionsFromTargetOptions(JTMB.getOptions())),
         JTMB(std::move(JTMB)) {
-    Compiler = tpde_llvm::LLVMCompiler::create(JTMB.getTargetTriple());
     assert(Compiler != nullptr && "Unknown architecture");
   }
 
@@ -50,11 +49,14 @@ public:
   }
 
 private:
-  std::unique_ptr<tpde_llvm::LLVMCompiler> Compiler;
+  static thread_local std::unique_ptr<tpde_llvm::LLVMCompiler> Compiler;
   std::vector<std::unique_ptr<std::vector<uint8_t>>> Buffers;
   JITTargetMachineBuilder JTMB;
 };
 
+thread_local std::unique_ptr<tpde_llvm::LLVMCompiler> TPDECompiler::Compiler =
+    tpde_llvm::LLVMCompiler::create(Triple(LLVM_HOST_TRIPLE));
+
 int main(int argc, char *argv[]) {
   InitLLVM X(argc, argv);
   cl::ParseCommandLineOptions(argc, argv, "TPDE ORC JIT Compiler\n");
```

We should also guard access to our underlying buffers:
```diff
@@ -35,15 +35,19 @@ public:
   }
 
   Expected<std::unique_ptr<MemoryBuffer>> operator()(Module &M) override {
-    Buffers.push_back(std::make_unique<std::vector<uint8_t>>());
-    std::vector<uint8_t> *B = *Buffers.back().get();
+    std::vector<uint8_t> *B;
+    {
+      std::lock_guard<std::mutex> Lock(BuffersAccess);
+      Buffers.push_back(std::make_unique<std::vector<uint8_t>>());
+      B = Buffers.back().get();
+    }
 
     if (!Compiler->compile_to_elf(M, *B)) {
       errs() << "Falling back to LLVM for module: " << M.getName() << "\n";
@@ -50,6 +54,7 @@ public:
 private:
   static thread_local std::unique_ptr<tpde_llvm::LLVMCompiler> Compiler;
   std::vector<std::unique_ptr<std::vector<uint8_t>>> Buffers;
+  std::mutex BuffersAccess;
   JITTargetMachineBuilder JTMB;
 };

```

Finally, we can switch on concurrent compilation:
```diff
@@ -27,6 +27,10 @@ static cl::opt<std::string> EntryPoint("entrypoint",
                                       cl::desc("Entry point function name"),
                                       cl::init("main"));

+static cl::opt<unsigned>
+    Threads("par", cl::desc("Compile csmith code on N threads concurrently"),
+            cl::init(1));
+
class TPDECompiler : public IRCompileLayer::IRCompiler {
public:
@@ -65,6 +65,8 @@ int main(int argc, char *argv[]) {
       -> Expected<std::unique_ptr<IRCompileLayer::IRCompiler>> {
     return std::make_unique<TPDECompiler>(JTMB);
   };
+  Builder.SupportConcurrentCompilation = true;
+  Builder.NumCompileThreads = Threads;
   std::unique_ptr<LLJIT> JIT = ExitOnErr(Builder.create());
 
   ThreadSafeModule TSM(std::move(Mod), std::move(Context));
```

### Exercise concurrent lookup

It needs a lot more support code to actually exercise concurrent compilation and do basic performance measurments. The [sample project on GitHub](https://github.com/weliveindetail/tpde-orc){:target="_blank"} has one possible implementation. Essentially, it loads a large self-contained module that I generated with [csmith](https://github.com/csmith-project/csmith){:target="_blank"} and adds 100 duplicates of it to the JIT with different entry-points. Then it issues a single JIT lookup for all the entry-points at once. A simplified version of the implementation could look like this:
```cpp
SymbolMap SymMap;
SymbolLookupSet EntryPoints = addDuplicates(JIT, Mod);

outs() << "Compiling " << EntryPoints.size() << " modules on " << Threads
        << " threads in parallel\n";

using namespace std::chrono;
auto ES = JIT->getExecutionSession();
auto SO = makeJITDylibSearchOrder({JIT->getMainJITDylib()});
auto Start = steady_clock::now();
{
  // Lookup all entry-points at once to execise concurrent compilation
  SymMap = ExitOnErr(ES.lookup(SO, EntryPoints));
}
auto End = steady_clock::now();
auto Elapsed = duration_cast<milliseconds>(End - Start);

outs() << "Compile-time was: " << Elapsed.count() << " ms\n";
```

This brings compile-times of the example down from ~2200ms to ~740ms when running on up to 8 threads in parallel:
```
> ./tpde-orc --par 8 tpde-orc/03-csmith-tpde.ll
Load module: tpde-orc/03-csmith-tpde.ll
Compiling 100 modules on 8 threads in parallel
...
Compile-time was: 737 ms
```

### Et voilà!

Let's take a break and look at the amount of complexity that LLVM handles in our little example! We parse a [well-defined human-readable representation](https://llvm.org/docs/LangRef.html){:target="_blank"} of turing-complete programs that can be generated from various general-purpose languages like C++, Fortran, Rust, Swift, Julia and Zig.

We add parsed modules into a composable JIT engine that identifies symbols and dependencies for us. It compiles our modules into native object code for various platforms and CPU architectures, while we can define an optimization pipeline of our choice and inject a custom code generator like TPDE. The engine then links the object code into an executable form without any platform-specific dynamic-library tricks! All of this happens in-memory and without external tools. It's really impressive that we can simply tell it to run on N threads in parallel and it just works! :)

Given that serious amount of complexity, one can imagine that there are some rabbit holes left to explore. I leave one open as an exercise to the interested reader: The [current implementation of ORC's `DynamicThreadPoolTaskDispatcher`](https://github.com/llvm/llvm-project/blob/release/20.x/llvm/lib/ExecutionEngine/Orc/TaskDispatch.cpp#L67){:target="_blank"} spawns a new thread for each job and doesn't reuse them. For our example this means `#threads == #jobs`, so we are back to `O(#jobs)` overhead with our thread-local `tpde_llvm::LLVMCompiler`. Maybe we can make a task dispatcher with an actual thread-pool for this purpose? If that sounds interesting and you make a pull-request, [ping me](https://github.com/weliveindetail){:target="_blank"} and I am happy to review it!

