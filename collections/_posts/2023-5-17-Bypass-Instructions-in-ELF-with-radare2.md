---
layout: article
title: Bypass Instructions in ELF with radare2
categories: Programming
tags: linux elf
eyeCatcher: https://sdtelectronics.github.io/assets/gallery/2023-5-17-Bypass-Instructions-in-ELF-with-radare2.jpg
abstract: Is it possible to take fine-grained control on a program after it being compiled? Directly manipulating the machine code in the binary can be an approach. As impractical as it sounds, this technique can be used to tackle real-world problems, as shown in this article.
---

## Motivation
To change the behavior of a compiled program, the typical approach is to edit the source code and rebuild the project. However, for small changes in a large code base, this is probably not the favorable way. Furthermore, the source code is simply not available in many cases, so it's impossible to rebuild the program. These are common reasons one may want to edit the machine code inside a ELF binary. On the other hand, ELF binaries are usually viewed as a black box filled with magic, which normal people are afraid to touch. This attitude is reasonable as ELF format is not designed to be human readable, which also makes editing ELF files non-trivial. In this article, we will see how key secrets inside a ELF can be unveiled with specific tools, and make editing an ELF approachable.

The technique described by this article was devised when solving a real-world problem. Let's use that as the example: After an update, the main component `dart` in [dart-sdk](https://github.com/dart-lang/sdk), the development toolkit of dartlang, was [reported incompatible with WSL1](https://github.com/dart-lang/sdk/issues/46749). It turned out that an optimization commit triggered a [bug](https://github.com/microsoft/WSL/issues/7569) in WSL, which is no longer actively maintained. As a fix from Microsoft probably won't happen, the only viable option is to revert that commit. As the change brought by this commit is quite small, it's a good candidate to be handled with binary editing.

## Methodology
As the function causing the problem is already known from the commit, the first step is to locate this function in the binary. In ELF files, names of functions are abstracted as symbols, and they are stored together with their metadata in a symbol table. As static symbol tables of released binaries are typically stripped to reduce the size, one may have no luck with `nm` to extract symbols from the binary. Fortunately, to enable dynamic linking and stack trace, a dynamic symbol table is typically preserved, and it can be dumped with `readelf`:

``` bash
readelf -sC --wide dart-sdk/bin/dart
```

This will show all content in the symbol table, producing an insanely long output. With the `C` flag, all dumped symbols are already demangled, so they can be directly filtered by `grep` with the original function name. The result is promising:

``` c++
11681: 000000000221cee0   138 FUNC    GLOBAL DEFAULT   14 dart::bin::File::Map(dart::bin::File::MapType, long, long, void*)
```

What we care is the offset in the second column, indicating the position where the function resides in the binary. With the full name of the function, the disassembly of the function can also be viewed with ease:

``` bash
objdump -dC --disassemble="dart::bin::File::Map(dart::bin::File::MapType, long, long, void*)" dart-sdk/bin/dart
```

From the disassembly，an easy fix can be derived: Simply remove a `mov` instruction. This sounds easy enough to be done with a hex editor, but when you open the ELF with the editor and jump to the address found by `readelf`, the result can be surprising: Bytes at that offset don't match the output of `objdump`.

This is where things getting complicated: With little-endian byte order, the hex sequence is twisted when the elf is viewed by a hex editor. Even though the reordered bytes can be decoded with some patience, you also have to take this mapping into account when making changes to the binary, which is surely very error-prone. Fortunately, there are dedicated tools to facilitate these hacks on binary. For instruction-level modification, `radare2` can be a good choice.

The main interface of `radare2` in an interactive shell. It will show up after the command is launched with the file to work on:

``` bash
radare2 -w dart-sdk/bin/dart
```

As we want to use `radare2` for editing, it's necessary to open the binary with the writing flag `-w`.

The first step is to analyze the file, with the command `aaaa`. This may take a while to complete. After that, jump to the address of the function where the problem is identified:
``` bash
s 0x221cee0
```

Use command `Vpc` to view the assembly of that function. Then, move the cursor with arrow keys to select the instruction to be edited, and press `w` `o` `a` to enter the edit mode. Radare supports a [set of substitutions](https://r2wiki.readthedocs.io/en/latest/options/w/wao-op/) for instructions on X86 and ARM platforms. For skipping a single instruction, `nop` does the job here. After entering the instruction, press `Esc` to quit the edit mode and use command `q` to save and exit.

## Conclusion
Instruction-level patching can be a powerful technique to change the behavior of a program, yet being hard to apply in practice. By tackling a real-world problem with this technique, the entire process from locating the function to replacing the instruction is demonstrated as above. With the same process, it's possible to perform modifications other than skip an instruction, offering a tool with great flexibility for development and hacking.
