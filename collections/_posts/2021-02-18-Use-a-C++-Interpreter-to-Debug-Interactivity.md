---
layout: article
title: Use a C++ Interpreter to Debug Interactivity
categories: Programming
tags: C++ debugging toolbox
eyeCatcher: assets/gallery/2021-02-18-head-cling.png
---

C++ is widely known as a statically compiled language. It enjoys the top tier performance but also suffers from great difficulty in debugging due to this reason. You may never expect there is an interpreter for C++ which allows dynamic execution like script languages such as Python, but such an amazing thing does exist. Here is the main character today: [cling](https://root.cern/cling/) the C++ interpreter developed by [Root](https://root.cern/) Team of CERN on the top of LLVM and Clang. You may have played with some toy-like C interpreter already, but cling is a sophisticated project with all features a modern REPL for a regular script language is supposed to have, such as stack trace, variable shadowing and statement value echo back.

Developers with experience writing script languages must have felt that how useful a REPL is when exploring language features and debugging modular code. As the interpreter for an compiled language, cling can do something even beyond a typical REPL can do, such as loading binary libraries. Hold your breath and take a closer look together!

## Quick Start
### hello world:
```C++
[cling]$ #include <stdio.h>
[cling]$ 
[cling]$ printf("hello world\n");
hello world
```
 You can include all headers contained in the search path of your system, including kernel headers. All symbols are ready to use after the include statement.

 ### Print Evaluated Values of Statements
 What if you omit the semicolon at the end of a statement? Will you get an error? No, by this cling will print the value of that statement. People have worked with matlab may feel familiar with this: If the ending semicolon isn't omitted, the echo in the console will be disabled, otherwise the evaluated value will be printed:
```C++
[cling]$ __cplusplus
(long) 201703
```
Cling can deserialize some objects and their references automatically. The supported objects are mainly STL containers:
```C++
[cling]$ #include <string>
[cling]$ 
[cling]$ std::string {"str"}
(std::string) "str"
[cling]$ 
[cling]$ #include <vector>
[cling]$ 
[cling]$ std::vector<char> ve{'c', 'h', 'a', 'r'} 
(std::vector<char> &) { 'c', 'h', 'a', 'r' }
[cling]$ 
[cling]$ #include <map>
[cling]$ 
[cling]$ std::map<int, char> mp{std::pair<int,char>()}
(std::map<int, char> &) { 0 => '0x00' }
```
Names of functions are also statements. What values will be echoed for them?
```C++
[cling]$ atoi
(int (*)(const char *) throw()) Function @0xb6c7bcd1
  at /usr/include/stdlib.h:104:
extern int atoi (const char *__nptr)
     __THROW __attribute_pure__ __nonnull ((1))
```
It's the signature of the function!

I enabled RTTI in my build script, so the dynamic inspection of types by typeid is also supported:
```C++
[cling]$ auto b = std::string ("typed")
(std::basic_string<char, std::char_traits<char>, std::allocator<char> > &) "typed"
[cling]$ typeid(b).name()
(const char *) "NSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE"
```
To convert this mangled type name to readable text, you may wish to use `c++filt`:
```C++
c++filt _ZNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >
```
## Take a Break and Get Ready for Brain Storming
Cling is one of the most imaginative projects I have ever seen. How useful it is depends on how creative you are. But here I'd like to specially address on its usage on embedded development. Accessing peripherals on a SoC is essentially wrapped by system calls, and only language can handle this easily is C. The compatibility with C of cling makes it able to execute system calls directly, which essentially implies you can do anything (allowed in user space) with it, dynamically and interactively. Granted, there are frameworks like microPython which allows you to interact with peripherals dynamically as well, but this is limited to those APIs already wrapped by the framework. In comparison, you can do anything a C program can do in user space with cling. Additionally, some performance-sensitive applications are preferably written in C/C++, and a REPL is desirable for fast-prototyping and debugging.

The text wall is tiresome. Let's see an example demonstrating interact with the new GPIO API, libgpiod, dynamically with cling:

![libgpiod](https://cdn.hackaday.io/images/1792991613815077908.png)

 Note that the official builds for cling are only available for x86/64 platforms currently, however, I managed to build it for armhf (arm v7) and aarch64 architecture with a QEMU+chroot environment. Here is the repository containing the build script and a pre-compiled binary in release:

 [github.com/SdtElectronics/cling-build](https://github.com/SdtElectronics/cling-build)

![scheme](https://z3.ax1x.com/2021/11/17/I5oHbt.png)

USB Tethering and network filesystem can be used together to amplify the convenience brought by
Cling to a whole new level. Thorough USB Tethering, the debugging host and slave can share network together, and communications with the slave can be established by SSH. By mounting network filesystem on the host, code in the 
storage of the host can be loaded directly by Cling on the slave without downloading[8]
. This
effectively eliminates all the compilation and downloading process in debugging required by the
conventional approach. Furthermore, storing source files on the host allows editing the code with
feature-rich editors like Visual Studio Code.


## More Advanced Usages
### Loading Files
Except standard C++ syntax, cling supports some internal commands as well. These commands are started with a dot, and you may have already used one of them to quit cling: `.q`. 

Let's see a command supporting load of external source files: `.L`
```C++
# echo 'int val = 42;' > tst.cpp
# ./cling
****************** CLING ******************
* Type C++ code and press enter to run it *
*             Type .q to exit             *
*******************************************
[cling]$ 
[cling]$ .L tst.cpp
[cling]$ val
(int) 42
```
The symbols in loaded files are ready to use now. Notice that cling can NOT handle the dependency of source files loaded for you (it don't know where the dependencies are, apparently), thus if the loaded files depend on other files, you need to load them manually.

A similar command is `.x`. It loads the designated file as well, however, if a function inside has the same name as the file, that function will be called immediately after loading:
```C++
# echo 'int tst(){return 42;}' > tst.cpp
# ./cling
****************** CLING ******************
* Type C++ code and press enter to run it *
*             Type .q to exit             *
*******************************************
[cling]$ 
[cling]$ .x tst.cpp
(int) 42
```
 One of the incredible features of cling is it can also load shared libraries (.so)! Notice that you'll need to include the corresponding headers to make the symbols in them accessible:
 ```C++
[cling]$ #include <gpiod.hpp>
[cling]$ 
[cling]$ .L /usr/lib/arm-linux-gnueabihf/libgpiod.so.2
[cling]$ 
[cling]$ .L /usr/lib/arm-linux-gnueabihf/libgpiodcxx.so
[cling]$ 
[cling]$ gpiod::chip chip("/dev/gpiochip0")
(gpiod::chip &) @0xb6fa100c
[cling]$ 
[cling]$ auto lines = chip.get_line(236)
(gpiod::line &) @0xb6fa1014
[cling]$ 
[cling]$ lines.request({.consumer = std::string("tst"),
[cling]$ ?      
[cling]$ ?      .request_type = gpiod::line_request::DIRECTION_OUTPUT,
[cling]$ ?      
[cling]$ ?      .flags = gpiod::line_request::FLAG_OPEN_DRAIN})
[cling]$
```
## Miscellaneous Commands
`.I` which adds or prints include searching paths:
 ```C++
[cling]$ .I
-cxx-isystem
/usr/include/c++/10
-cxx-isystem
/usr/include/arm-linux-gnueabihf/c++/10
-cxx-isystem
/usr/include/c++/10/backward
-isystem
/usr/local/include
-isystem
/home/cling-src/release/bin/../lib/clang/5.0.0/include
-extern-c-isystem
/usr/include/arm-linux-gnueabihf
-extern-c-isystem
/include
-extern-c-isystem
/usr/include
-I
/home/cling-src/release/include
-I
/home/cling-src/llvm/tools/cling/include
-I
.
-resource-dir
/home/cling-src/release/bin/../lib/clang/5.0.0
-nostdinc++
```
`.class`, a powerful command which prints the layout of  specified class:
```C++
[cling]$ #include<limits>
[cling]$ 
[cling]$ .class std::numeric_limits<char>
===========================================================================
struct std::numeric_limits<char>
SIZE: 1 FILE: limits LINE: 453
List of member variables --------------------------------------------------
limits          455 0x0      public: static constexpr _Bool is_specialized
limits          468 0x0      public: static constexpr int digits
limits          469 0x0      public: static constexpr int digits10
limits          471 0x0      public: static constexpr int max_digits10
limits          473 0x0      public: static constexpr _Bool is_signed
limits          474 0x0      public: static constexpr _Bool is_integer
limits          475 0x0      public: static constexpr _Bool is_exact
limits          476 0x0      public: static constexpr int radix
limits          484 0x0      public: static constexpr int min_exponent
limits          485 0x0      public: static constexpr int min_exponent10
limits          486 0x0      public: static constexpr int max_exponent
limits          487 0x0      public: static constexpr int max_exponent10
limits          489 0x0      public: static constexpr _Bool has_infinity
limits          490 0x0      public: static constexpr _Bool has_quiet_NaN
limits          491 0x0      public: static constexpr _Bool has_signaling_NaN
limits          492 0x0      public: static constexpr enum std::float_denorm_style has_denorm
limits          494 0x0      public: static constexpr _Bool has_denorm_loss
limits          508 0x0      public: static constexpr _Bool is_iec559
limits          509 0x0      public: static constexpr _Bool is_bounded
limits          510 0x0      public: static constexpr _Bool is_modulo
limits          512 0x0      public: static constexpr _Bool traps
limits          513 0x0      public: static constexpr _Bool tinyness_before
limits          514 0x0      public: static constexpr enum std::float_round_style round_style
List of member functions :---------------------------------------------------
filename     line:size busy function type and name
(compiled)     (NA):(NA) 0 public: static constexpr char min() noexcept;
(compiled)     (NA):(NA) 0 public: static constexpr char max() noexcept;
(compiled)     (NA):(NA) 0 public: static constexpr char lowest() noexcept;
(compiled)     (NA):(NA) 0 public: static constexpr char epsilon() noexcept;
(compiled)     (NA):(NA) 0 public: static constexpr char round_error() noexcept;
(compiled)     (NA):(NA) 0 public: static constexpr char infinity() noexcept;
(compiled)     (NA):(NA) 0 public: static constexpr char quiet_NaN() noexcept;
(compiled)     (NA):(NA) 0 public: static constexpr char signaling_NaN() noexcept;
(compiled)     (NA):(NA) 0 public: static constexpr char denorm_min() noexcept;
```
Finally, ` .help` command which prints all available (not really, some commands hinted in the ROOT prompt seem to be omitted here) commands and their corresponding usages:
```C++
[cling]$ .help

 Cling (C/C++ interpreter) meta commands usage
 All commands must be preceded by a '.', except
 for the evaluation statement { }
 ==============================================================================
 Syntax: .Command [arg0 arg1 ... argN]

   .L <filename>                - Load the given file or library

   .(x|X) <filename>[args]      - Same as .L and runs a function with
                                  signature: ret_type filename(args)

   .> <filename>                - Redirect command to a given file
      '>' or '1>'               - Redirects the stdout stream only
      '2>'                      - Redirects the stderr stream only
      '&>' (or '2>&1')          - Redirects both stdout and stderr
      '>>'                      - Appends to the given file

   .undo [n]                    - Unloads the last 'n' inputs lines

   .U <filename>                - Unloads the given file

   .I [path]                    - Shows the include path. If a path is given -
                                  adds the path to the include paths

   .O <level>                   - Sets the optimization level (0-3)
                                  (not yet implemented)

   .class <name>                - Prints out class <name> in a CINT-like style

   .files                       - Prints out some CINT-like file statistics

   .fileEx                      - Prints out some file statistics

   .g                           - Prints out information about global variable
                                  'name' - if no name is given, print them all

   .@                           - Cancels and ignores the multiline input

   .rawInput [0|1]              - Toggle wrapping and printing the
                                  execution results of the input

   .dynamicExtensions [0|1]     - Toggles the use of the dynamic scopes and the
                                  late binding

   .printDebug [0|1]            - Toggles the printing of input's corresponding
                                  state changes

   .storeState <filename>       - Store the interpreter's state to a given file

   .compareState <filename>     - Compare the interpreter's state with the one
                                  saved in a given file

   .stats [name]                - Show stats for internal data structures
                                  'ast'  abstract syntax tree stats
                                  'asttree [filter]'  abstract syntax tree layout
                                  'decl' dump ast declarations
                                  'undo' show undo stack

   .help                        - Shows this information

   .q                           - Exit the program
```
