NativeJIT
====

NativeJIT is an open-source cross-platform library for high-performance
just-in-time compilation of expressions involving C data structures.
The compiler is light weight and fast
and it takes no dependencies beyond the standard C++ runtime.
It runs on Linux, OSX, and Windows.
The generated code is optimized with particular attention paid
to register allocation.

The compiler was developed by the [Bing](http://www.bing.com/) team for use in the Bing search engine.
One important use is scoring documents containing keywords that match a user's query.
The scoring process attempts to gauge how well each document matches the user's intent,
and as such, depends on the specifics of how each query was phrased.
Bing formulates a custom expression for each query
and then uses NativeJIT to compile the expression into x64 code that will
be run on a large set of candidate documents spread across a cluster of
machines.

We knew from the get go that throughput and latency
would be essential when processing queries at scale,
so we put a lot of effort in to making NativeJIT run fast.

Our design point was scenarios where

* The expression isn't known until runtime.
* The expression will be evaluated enough times to amortize the cost of compilation.
* Latency and throughput demands require low cost for compilation.


Here's trivial "Hello, world" level example that computes the area of a circle:

~~~
#include "NativeJIT/CodeGen/ExecutionBuffer.h"
#include "NativeJIT/CodeGen/FunctionBuffer.h"
#include "NativeJIT/Function.h"
#include "Temporary/Allocator.h"

#include <iostream>

using NativeJIT::Allocator;
using NativeJIT::ExecutionBuffer;
using NativeJIT::Function;
using NativeJIT::FunctionBuffer;

int main()
{
    // Create allocator and buffers for pre-compiled and post-compiled code.
    ExecutionBuffer codeAllocator(8192);
    Allocator allocator(8192);
    FunctionBuffer code(codeAllocator, 8192);

    // Create the factory for expression nodes.
    // Our area expression will take a single float parameter and return a float.
    Function<float, float> expression(allocator, code);

    // Multiply input parameter by itself to get radius squared.
    auto & rsquared = expression.Mul(expression.GetP1(), expression.GetP1());

    // Multiply by PI.
    const float  PI = 3.14159265358979f;
    auto & area = expression.Mul(rsquared, expression.Immediate(PI));

    // Compile expression into a function.
    auto function = expression.Compile(area);

    // Now run our expression!
    float radius = 2.0;
    std::cout << "The area of a circle with radius " << radius
    	      << " is " << function(radius);
    	      
    return 0;
}
~~~

Here is the generated assembly code on Windows (NOTE: there seems to be a bug in the
register allocator. While the code computes the correct result, it is less efficient
than it would have been if the entire result had been computed directly
into xmm0, without ever using xmm15).

~~~
PI_CONSTANT:
    db 0f 49 40                              ; PI constant is stored in memory.
ENTRY_POINT:
    sub    rsp, 28h                          ; Standard function prolog
    mov    qword ptr[rsp], rbp
    movaps xmmword ptr[rsp + 10h], xmm15
    lea    rbp, [rsp + 28h]

    movss  xmm15, xmm0                       ; Load radius from first parameter register.
    mulss  xmm15, xmm0                       ; Multiply by radius.
    mulss  xmm15, dword ptr[PI_CONSTANT]     ; Multiply by PI.
    movss  xmm0, xmm15                       ; Return value goes in xmm0.

    movaps xmm15, xmmword ptr[rsp + 10h]     ; Standard function epilog.
    mov    rbp, qword ptr[rsp]
    add    rsp, 28h
    ret
~~~


This example shows an expression that multiplies a number by itself.
We also support a wide variety of arithmetic and logical operations, pointer and array operations, conditionals, accessing structure fields, and calling out to C functions.
[See our preliminary API docs in `/Documentation` for more information](https://github.com/BitFunnel/NativeJIT/tree/master/Documentation).
Coming soon: better online documentation at [bitfunnel.org](https://github.com/bitfunnel/nativejit).


Dependencies
------------

In order to build NativeJIT you will need CMake (2.8.11+), and a modern C++
compiler (gcc 5+, clang 3.4+, or VC 2015+).

Following are specific instructions for building on various platforms.

### Linux

NativeJIT can be built with Clang (3.4+) or GNU C++ (5.0+). We recommend the
latest version of Clang as it's much faster with template-heavy code like
NativeJIT.

Run the following commands to install the minimal set of packages needed to 
build the core NativeJIT library on Ubuntu 15 or 16:

~~~
sudo apt-get install clang cmake
~~~

If you're on an older version of Ubuntu, like 14, you'll need to install a newer version of CMake. To
install a new-enough CMake, see [this link](http://askubuntu.com/questions/610291/how-to-install-cmake-3-2-on-ubuntu-14-04).
If you're using gcc, you'll also need to make sure you have gcc-5 (`sudo apt-get install g++-5`).

In the root `NativeJIT` directory run:

~~~
./Configure_Linux_Make_clang.sh
cd build-linux-clang
make
~~~

### OSX

Install XCode and then run the following command to install required packages 
using Homebrew ([http://brew.sh/](http://brew.sh/)):

~~~
brew install cmake
~~~

NativeJIT can be built on OS X using either standard \*nix makefiles or XCode.
In order to generate and build makefiles, in the root `NativeJIT` directory run:

~~~
./Configure_OSX_Make.sh
~~~
    
Alternatively you can generate XCode project by running

~~~
./Configure_OSX_XCode.sh
~~~

You can build and run unit tests by building the `test` target in XCode or by
running make in the build directory:

~~~
make test
~~~

### Windows

Install the following tools:

- Visual Studio 2015 with C++ compiler
- CMake ([http://www.cmake.org/download/](http://www.cmake.org/download/))

You can get [the free version of Visual Studio here](https://www.visualstudio.com/en-us/products/visual-studio-community-vs.aspx).
Note that if you're installing Visual Studio for the first time and select the
default install options, you won't get a C++ compiler. To force the install of
the C++ compiler, you need to either create a new C++ project or open an
existing C++ project.

In order to configure solution for Visual Studio 2015 run the following 
commands from the root `NativeJIT` directory:

~~~
.\Configure_Windows_MSVC.bat
~~~

From 
now on you can use the generated solution `build-win-msvc\BitFunnel.sln` from Visual Studio
or build from command line using `cmake`:

