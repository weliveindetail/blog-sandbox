---
layout: post
categories: post
author: Stefan Gränitz
date: 2023-10-19 10:00:00 +0200
image: https://weliveindetail.github.io/blog/res/2023-llvm-repo-structure.png
preview: summary_large_image
title: "Maintain a reduced fork of LLVM"
description: "What about stripping down the LLVM monorepo in order to checkout and build one specific tool quickly? It’s pretty simple to do once. This post describes how to keep it maintainable."
source: https://github.com/weliveindetail/blog/main/_posts/2023-10-19-llvm-jitlink-executor.md
---

<style>
  #large-image {
    max-width: min(100%, 400px);
  }
  .center {
    display: block;
    margin: 0 auto;
  }
</style>

The [LLVM monorepo](https://llvm.org/docs/DeveloperPolicy.html#adding-an-established-project-to-the-llvm-monorepo){:target="_blank"} turned into a beast. At the time of writing this post, a fresh checkout of the `main` branch is 3.9G in size and it will only keep growing.

The massive size of the repository can make our lifes as developers a little uncomfortable at times and it inflates network traffic for build bots. It's a real problem if we want to build a tiny piece from LLVM on a small device. This is the case with the `llvm-jitlink-executor` tool, which is the RPC endpoint for ORC JIT in upstream LLVM. It's very useful when developing new LLVM JITLink backends like AArch32, because devices with this CPU architecture are no suitable development machines. We cannot checkout and build LLVM on a Raspberry Pi. It just doesn't have enough resources. Instead we want to continue developing on our favored workstation (like a regular Linux PC), cross-JIT compile our test code and only execute it on the remote device. Sure, we could cross-compile LLVM for the target device on our workstation and transfer just the `llvm-jitlink-executor` binary, but we'd have to set up a proper cross-toolchain and that is quite some effort too. It's much easier to checkout a small repository on the device itself and build the binary there.

The [`llvm-jitlink-executor` tool itself is fairly small](https://github.com/llvm/llvm-project/tree/release/17.x/llvm/tools/llvm-jitlink/llvm-jitlink-executor){:target="_blank"} and it only depends on the `Support`, `OrcShared` and `OrcTargetProcess` libraries. We don't need any other subproject from the monorepo and we don't want to develop on the device, so we don't need the git history either. All that's necessary is the matching source version, because the RPC interface doesn't provide a versioning mechanism. What about stripping down the LLVM monorepo and cutting down the git history so that we can checkout and build the `llvm-jitlink-executor` tool quickly? It's pretty simple when we do it once. This post describes how to keep it maintainable.

### Mapping to the LLVM upstream repository

Maintaining repositories downstream from LLVM is a challenge on its own, because LLVM develops quickly and doesn't care much about backwards compatibility. Our scenario is simplified though, because we have no downstream changes, which are modifications of the upstream code somewhere in the middle of the git history. We basically only want snapshots of the LLVM monorepo at specific points in history that are smaller in size. The release branch structure should follow the one in LLVM:

![llvm-release-branches](https://weliveindetail.github.io/blog-sandbox/res/2023-llvm-repo-structure.png){: #large-image}{: .center}

The last commit before a branch point can be found with `git merge-base`:
```
llvm-project ➜ git remote -v
origin      https://github.com/llvm/llvm-project (fetch)
origin      <invalidated> (push)
llvm-project ➜ git merge-base release/16.x main
b0daacf58f417634f7c7c9496589d723592a8f5a
llvm-project ➜ git merge-base release/17.x main
d0b54bb50e5110a004b41fc06dadf3fee70834b7
```

`16.x` branched after `b0daacf58f41` and `17.x` branched after `d0b54bb50e51`. While release branches are reasonably short, the number of commits in between releases can be huge:
```
llvm-project ➜ git log --oneline b0daacf58f41..release/16.x | wc -l
     315
llvm-project ➜ git log --oneline d0b54bb50e51..release/17.x | wc -l
     342
llvm-project ➜ git log --oneline b0daacf58f41..d0b54bb50e51 | wc -l
   19305
```

Do we need all this history? Not really, a few snapshots would be sufficient. In principle we could store them as unrelated `--orphan` branches, but with a growing number of versions we would accumulate lots of isolated blobs that don't share any data. It would be better to keep the `--orphan` branch for our first version and continue with a reduced history where we squash most patches into a few single commits:
```
llvm-jitlink-executor ➜ git log --oneline main
0c846d6b14e1 [downstream] Add README and extra debug dumps
c0ff441b55ee 0e1d7239d6fd Mainline increment release/18.x (19 Oct 2023)
405c669d60ca 77c923bb7aea Mainline increment release/17.x
535d946540a6 4c7f53b99c08 [Orc] Add AutoRegisterCode option for DebugObjectManagerPlugin

llvm-jitlink-executor ➜ git log --oneline release/17.x
0a0e397f03fa [downstream] Add README and extra debug dumps
3bffe7c2a561 0ebc2997b74a Release branch increment release/17.x
405c669d60ca 77c923bb7aea Mainline increment release/17.x
535d946540a6 4c7f53b99c08 [Orc] Add AutoRegisterCode option for DebugObjectManagerPlugin
^            ^
Commit ID    Upstream ID
```

It's a good idea to add the ID of the latest included upstream commit as a prefix to the log message in each squashed commit. This way we know which source version it represents upstream without having to grep through history.

### Let's build it in git!

Last time I updated the repo was for my [2023 EuroLLVM presentation](https://www.youtube.com/watch?v=9jFXNRzDSf0){:target="_blank"}. This is where we start from:
```
llvm-jitlink-executor ➜ git remote -v
origin      https://github.com/echtzeit-dev/llvm-jitlink-executor (fetch)
origin      https://github.com/echtzeit-dev/llvm-jitlink-executor (push)
upstream    https://github.com/llvm/llvm-project (fetch)
upstream    <invalidated> (push)
llvm-jitlink-executor ➜ git status
On branch eurollvm-2023
Your branch is up to date with 'origin/eurollvm-2023'.

nothing to commit, working tree clean
llvm-jitlink-executor ➜ git log --oneline
535d946540a6 4c7f53b99c08 [Orc] Add AutoRegisterCode option for DebugObjectManagerPlugin
```

We checkout the branch point for `release/17.x` and squash all commits since `4c7f53b99c08`:
```
llvm-jitlink-executor ➜ git fetch upstream release/17.x main
llvm-jitlink-executor ➜ git merge-base upstream/release/17.x upstream/main
d0b54bb50e5110a004b41fc06dadf3fee70834b7
llvm-jitlink-executor ➜ git checkout -b release/17.x-increment d0b54bb50e51
llvm-jitlink-executor ➜ ls
CODE_OF_CONDUCT.md  compiler-rt         llvm
CONTRIBUTING.md     cross-project-tests llvm-libgcc
LICENSE.TXT         flang               mlir
README.md           libc                openmp
SECURITY.md         libclc              polly
bolt                libcxx              pstl
build               libcxxabi           runtimes
clang               libunwind           third-party
clang-tools-extra   lld                 utils
cmake               lldb

llvm-jitlink-executor ➜ git reset --soft 4c7f53b99c08
llvm-jitlink-executor ➜ git commit -m "Mainline increment release/17.x"
```

We can do the same for the commits on the release branch and get the reduced history upstream:
```
llvm-jitlink-executor ➜ git checkout -b release/17.x-fixes upstream/release/17.x
llvm-jitlink-executor ➜ git reset --soft release/17.x-increment
llvm-jitlink-executor ➜ git commit -m "Release branch increment release/17.x"
llvm-jitlink-executor ➜ git log --oneline -3
0ebc2997b74a (HEAD -> release/17.x-fixes) Release branch increment release/17.x
77c923bb7aea (release/17.x-increment) Mainline increment release/17.x
4c7f53b99c08 [Orc] Add AutoRegisterCode option for DebugObjectManagerPlugin
```

Now let's bring them downstream with `git cherry-pick`. We resolve conflicts by removing the subprojects we don't need and by adding everything else:
```
llvm-jitlink-executor ➜ git checkout eurollvm-2023
llvm-jitlink-executor ➜ git checkout -b main
llvm-jitlink-executor ➜ git cherry-pick --no-commit 77c923bb7aea
llvm-jitlink-executor ➜ rm -rf bolt clang clang-tools-extra cross-project-tests flang libc libclc libcxx libcxxabi libunwind lld lldb llvm-libgcc mlir openmp polly pstl runtimes utils
llvm-jitlink-executor ➜ git add .
llvm-jitlink-executor ➜ git commit -m "77c923bb7aea Mainline increment release/17.x
llvm-jitlink-executor ➜ git log --oneline
405c669d60ca (HEAD -> main) 77c923bb7aea Mainline increment release/17.x
535d946540a6 (eurollvm-2023) 4c7f53b99c08 [Orc] Add AutoRegisterCode option for DebugObjectManagerPlugin
```

We can do the same for the release branch:
```
llvm-jitlink-executor ➜ git checkout -b release/17.x
llvm-jitlink-executor ➜ git cherry-pick --no-commit 0ebc2997b74a
llvm-jitlink-executor ➜ rm -rf bolt clang clang-tools-extra cross-project-tests flang libc libclc libcxx libcxxabi libunwind lld lldb llvm-libgcc mlir openmp polly pstl runtimes utils
llvm-jitlink-executor ➜ git commit -m "0ebc2997b74a Release branch increment release/17.x"
llvm-jitlink-executor ➜ git log --oneline
3bffe7c2a561 (HEAD -> release/17.x) 0ebc2997b74a Release branch increment release/17.x
405c669d60ca (main) 77c923bb7aea Mainline increment release/17.x
535d946540a6 (eurollvm-2023) 4c7f53b99c08 [Orc] Add AutoRegisterCode option for DebugObjectManagerPlugin
```

Then make a smoke check that the executable still builds and publish the downstream branch:
```
llvm-jitlink-executor ➜ ls
CONTRIBUTING.md SECURITY.md     compiler-rt
LICENSE.TXT     build           llvm
README.md       cmake           third-party

llvm-jitlink-executor ➜ ninja -C build llvm-jitlink-executor
[0/1] Re-running CMake...
...
[221/221] Linking CXX executable bin/llvm-jitlink-executor
build ➜ ls -lh build/bin/llvm-jitlink-executor
-rwxr-xr-x  1 ez     staff   1.3M Oct 19 12:30 bin/llvm-jitlink-executor
llvm-jitlink-executor ➜ git push --set-upstream origin release/17.x
```

### More relevant changes on mainline?

The `llvm-jitlink-executor` tool doesn't depend on all of LLVM, but only a few pieces, in particular:
```
llvm/include/llvm/ExecutionEngine/Orc/Shared
llvm/include/llvm/ExecutionEngine/Orc/TargetProcess
llvm/lib/ExecutionEngine/Orc/Shared
llvm/lib/ExecutionEngine/Orc/TargetProcess
llvm/tools/llvm-jitlink/llvm-jitlink-executor
```

In order to see if there are recent changes that should be incorporated, we can check the log for these locations since the squash point:
```
llvm-jitlink-executor ➜ git log --oneline -1 77c923bb7aea
77c923bb7aea Mainline increment release/17.x
llvm-jitlink-executor ➜ git log --oneline 77c923bb7aea..upstream/main -- llvm/lib/ExecutionEngine/Orc/Shared
122ebe3b5001 [ORC] Use EPC bootstrap symbols to communicate eh-frame registration fn addrs.
e76ac8074f85 [llvm][orc] Consider other ELF init sections as well
f448d44663a5 [ORC][ORC-RT][MachO] Use _objc_(map|load)_images for ObjC & Swift registration.
```

We don't have a branch point for the next release yet, so we might want to count increments:
```
llvm-jitlink-executor ➜ git checkout -b release/18.x-increment01 upstream/main
llvm-jitlink-executor ➜ git reset --soft 77c923bb7aea
llvm-jitlink-executor ➜ git commit -m "Mainline increment release/18.x (19 Oct 2023)"
llvm-jitlink-executor ➜ git log --oneline -3
5a9500c73679 (HEAD -> release/18.x-increment01) Mainline increment release/18.x (19 Oct 2023)
77c923bb7aea (release/17.x-increment) Mainline increment release/17.x
4c7f53b99c08 [Orc] Add AutoRegisterCode option for DebugObjectManagerPlugin
```

We `git cherry-pick` the increment to our downstream `main` branch and apply the same conflict resolution as before:
```
llvm-jitlink-executor ➜ git checkout main
llvm-jitlink-executor ➜ git log --oneline
405c669d60ca (HEAD -> main) 77c923bb7aea Mainline increment release/17.x
535d946540a6 (eurollvm-2023) 4c7f53b99c08 [Orc] Add AutoRegisterCode option for DebugObjectManagerPlugin
llvm-jitlink-executor ➜ git cherry-pick --no-commit 5a9500c73679
llvm-jitlink-executor ➜ rm -rf bolt clang clang-tools-extra cross-project-tests flang libc libclc libcxx libcxxabi libunwind lld lldb llvm-libgcc mlir openmp polly pstl runtimes utils
llvm-jitlink-executor ➜ git add .
llvm-jitlink-executor ➜ git rev-parse upstream/main
0e1d7239d6fddebdaf39e58eb931ff4916306b23
llvm-jitlink-executor ➜ git commit -m "0e1d7239d6fd Mainline increment release/18.x (19 Oct 2023)"
llvm-jitlink-executor ➜ git log --oneline
c0ff441b55ee (HEAD -> main) 0e1d7239d6fd Mainline increment release/18.x (19 Oct 2023)
405c669d60ca 77c923bb7aea Mainline increment release/17.x
535d946540a6 (eurollvm-2023) 4c7f53b99c08 [Orc] Add AutoRegisterCode option for DebugObjectManagerPlugin
```

## Voila!

In the last step, we cherry-pick the commit that adjusts the README and adds extra debug dumps in `llvm-jitlink-executor` on top of each downstream branch. Let's smoke check a last time and publish our downstream `main` branch:
```
llvm-jitlink-executor ➜ ls
CONTRIBUTING.md SECURITY.md     compiler-rt
LICENSE.TXT     build           llvm
README.md       cmake           third-party
llvm-jitlink-executor ➜ ninja -C build llvm-jitlink-executor
[0/1] Re-running CMake...
...
[218/218] Linking CXX executable bin/llvm-jitlink-executor
build ➜ ls -lh build/bin/llvm-jitlink-executor
-rwxr-xr-x  1 ez  staff   1.3M Oct 19 16:57 bin/llvm-jitlink-executor
llvm-jitlink-executor ➜ git push --set-upstream origin main
```
