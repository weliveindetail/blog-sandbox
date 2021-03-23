---
layout: post
author: Stefan Gränitz
title:  "LLDB regained JITed code debugging feature"
date:   2021-03-01 16:07:01 +0200
categories: post
---

Today [LLVM 12 was officially released](https://releases.llvm.org/) and with it a new LLDB version that (finally!) regained source-level debug support through the [GDB JIT interface](https://sourceware.org/gdb/current/onlinedocs/gdb/JIT-Interface.html). It has been broken in releases 6 to 11 [due to a regression](https://llvm.org/PR36209) that involved two autonomous bugs.

The feature is now [continuously tested to work for ELF on x86-64](https://github.com/llvm/llvm-project/blob/release/12.x/lldb/test/Shell/Breakpoint/jitbp_elf.test). MachO and COFF don't seem to work and I didn't investigate the state on ARM and other architectures. If you want to give it a quick try without setting up the environment, there is a [Docker container here](https://hub.docker.com/repository/docker/weliveindetail/lldb-12-jit-debug-demo) that reproduces it reliably:

```
% docker run --privileged --cap-add=SYS_PTRACE --security-opt seccomp=unconfined --security-opt apparmor=unconfined \
             --rm -it weliveindetail/lldb-12-jit-debug-demo
(lldb) target create "lli-12"
Current executable set to 'lli-12' (x86_64).
(lldb) b jitbp
Breakpoint 1: no locations (pending).
WARNING:  Unable to resolve breakpoint to any actual locations.
(lldb) run /home/jitbp.ll
1 location added to breakpoint 1
Process 18 stopped
* thread #1, name = 'lli-12', stop reason = breakpoint 1.1
    frame #0: 0x00007ffff7fce004 JIT(0x4d0b50)`jitbp() at jitbp.cpp:1:15
-> 1   	int jitbp() { return 0; }
   2   	int main() { return jitbp(); }

Process 18 launched: '/usr/bin/lli-12' (x86_64)
(lldb) bt
* thread #1, name = 'lli-12', stop reason = breakpoint 1.1
  * frame #0: 0x00007ffff7fce004 JIT(0x4d0b50)`jitbp() at jitbp.cpp:1:15
    frame #1: 0x00007ffff7fce02b JIT(0x4d0b50)`main at jitbp.cpp:2:21
    frame #2: 0x00007ffff48f1aa7 libLLVM-12.so.1`llvm::MCJIT::runFunction(llvm::Function*, llvm::ArrayRef<llvm::GenericValue>) + 743
    frame #3: 0x00007ffff48a3c00 libLLVM-12.so.1`llvm::ExecutionEngine::runFunctionAsMain(llvm::Function*, std::vector<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >, std::allocator<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > > > const&, char const* const*) + 1120
    frame #4: 0x0000000000414e53 lli-12`main + 8963
```

Note that the additional docker options are required to allow debugging inside the container.
