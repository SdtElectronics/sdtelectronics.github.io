---
layout: article
title: Patch Binary to Use An Alternative Libc
categories: Environment
tags: linux elf libc
eyeCatcher: https://sdtelectronics.github.io/assets/gallery/2023-4-15-Patch-Binary-to-Use-An-Alternative-Libc.jpg
abstract: It's handy to instruct an executable to use a different version of libc when encountering some compatibility issues. By patching the binary, this change can be made permanent.
---

Windows Subsystem for Linux (WSL) has once shown a great potential to be an elegant solution for a Linux development environment seamlessly embedded in a Windows host. Sadly, Microsoft decided to drop the initial idea of native binary support and took the path of full virtualization in WSL2, which defeats its purpose of being a light-weight subsystem and leaves many problems that will never be solved. One of these problems eventually became a disaster to all users of rolling distros like Arch-Linux: The version of the kernel baked in the system stuck at 4.4.0, which was no longer supported by glibc higher than 2.33. This problem struck me about two years ago after a full update and suspended all programs dynamically linked to glibc (basically every program). It took me half a day to downgrade the libc externally (almost everything in the Linux environment broke) and revive the system. After that, the package manager became virtually unusable as all up-to-date programs link to the new version of libc. Despite how destructive it is, this problem didn't make much impact back then, presumably because many users have already migrated to WSL2 at that time. Being one of those stubborn minorites who decided to stick on WSL1, I have to fetch old packages from the archive to get them installed, which can quickly become a nightmare if it comes with lots of dependencies.

A few days ago, I finally decided to revisit this problem and see if there has been a real solution. Surprisingly, the brilliant developers of Arch WSL did come up with one. [One of Arch's unofficial repositories](https://repo.archlinuxcn.org/x86_64/) maintains a [newer glibc built specifically for 4.x kernels](https://aur.archlinux.org/packages/glibc-linux4), and can replace the libc from the official repo to work on WSL1. Unfortunately, this doesn't work for me, as there are already too many packages rely on the current glibc and replacing it breaks the dependency. So is there no redemption in this situation?

An approach I came up with is asking the executable to use a different glibc, and it turns out that this can be achieved by modifying the binary. There is a section in the elf binary of a dynamically linked program which instructs the linker where to find required libraries, and it can be modified by the tool `patchelf`:
``` bash
patchelf --set-interpreter $PATH_TO_LIBC/ld-linux.so.2 --set-rpath $PATH_TO_LIBC/ $PATH_TO_THE_EXECUTABLE
```

where `$PATH_TO_LIBC` points to the directory extracted from the download libc packages and contains `libc.so.6`, `libm.so.6` etc. There may be some variations in the name of the interpreter, and in my case, it's named as `ld-linux-x86-64.so.2`. The path to the executable, if it's not known as the software was obtained from the package manager, can be checked with the command `which`.

The advantage of this approach is that this modification is permanent and no special operation is need the next time executing the program. The downside also comes from the permanent modification: You'd better have the original executable backed up, in case this operation breaks the binary. So how can the binary be broken?

The answer is that some programs have its own `RPATH` to search for libraries located outside the system path. It will be overwritten if the above command is applied directly. This path can be retrieved with the command below:
``` bash
objdump -x $PATH_TO_THE_EXECUTABLE | grep 'R.*PATH'
```

If this program happens to have more than one search path, you would see them are delimited with a colon. Similarity, you can concatenate this path after the path of libc with a `:` in arguments of `patchelf`:
``` bash
patchelf --set-interpreter /opt/libc4/lib/ld-linux-x86-64.so.2 --set-rpath $PATH_TO_LIBC:$RETRIEVED_R_PATH
```

This method can be used in other scenario that an alternative libc is required to run a program. But of course, the most difficult part in that situation is to find a suitable libc release. If the version of the desired libc is already known, and the target is not some exotic architecture, the pre-built package may be found in the archive of mainstream distros, such as [Arch Linux Archive](https://wiki.archlinux.org/title/Arch_Linux_Archive#Historical_Archive). Don't copy the files directly to `/usr/lib` or you are at the risk of destroying the original system!
