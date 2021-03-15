---
layout: post
author: Stefan Gränitz
title:  "What's the matter with yaml-bench in the llvm-9-dev apt package?"
date:   2021-02-01 16:07:01 +0200
categories: post
---

The LLVM codebase is growing. It's a natural consequence for any successful library to grow over time. Inevitably it brings challenges



The above chart compares coverage stats for the [core LLVM libraries](https://github.com/llvm/llvm-project/tree/main/llvm) over the last 4 years. Bars show coverage in percent, lines show the total number of source lines and functions.

# Interpretation

The results indicate that overall code coverage remained at a high level, despite of a formidabel codebase growth of factor 1.6 during the survey period.

# Survey process

The data was collected by running the `check-llvm` test suite on a [gcov](https://man7.org/linux/man-pages/man1/gcov.1.html) instrumented build with all targets enabled. You can find the [script on GitHub](TODO).

```
    Line       Count   Code

     258          52 : void CompileOnDemandLayer::emitPartition(
     259             :     std::unique_ptr<MaterializationResponsibility> R, ThreadSafeModule TSM,
     260             :     IRMaterializationUnit::SymbolNameToDefinitionMap Defs) {
```

# Data

