zerostack: Zeroing stack and registers of sensitive functions
=============================================================
This project contains Clang/LLVM patches/passes to zero the stack and registers of sensitive functions. There are 
3 solutions implemented, as per the paper "What you get is what you C: Controlling side effects in mainstream C compilers", 
available [here](https://github.com/lmrs2/secretgrind). The code is a reference implementation, and is not supposed to be production ready. Nevertheless, 
this should be ebough for anyone who wants to test it.

The setup below was tested on Ubuntu trusty 14.04.5 LTS x86_64. I suggest you install this as a VM before reading further.

Pre-requesites:
---------------
	1. Install a VM running Ubuntu trusty 14.04.5 LTS x86_64. Allocate 32GB of disk.
	2. $sudo apt-get install git cmake g++ binutils-dev
	3. $export BASE_DIR=/whereever/you/want
	4. $cd $BASE_DIR

Download, compile and install Clang/LLVM (function-based implementation):
-----------------------------------------------------------
	$git clone https://github.com/lmrs2/llvm.git -b zerostack_38 --single-branch --depth 1 
	$cd llvm/tools
	$git clone https://github.com/lmrs2/clang.git -b zerostack_38 --single-branch --depth 1 
	$cd ..
	$mkdir build && cd build
	$cmake -DLLVM_BINUTILS_INCDIR=/usr/include -DBUILD_SHARED_LIBS=ON -DCMAKE_BUILD_TYPE=Debug -DLLVM_TARGETS_TO_BUILD="X86" ../
	$cmake --build .
	$sudo make install
	$export LLVM_SRC=`llvm-config --src-root`
	$export LLVM_BUILD=$LLVM_SRC/build

Download, compile and install compiler_rt:
-----------------------------------------
	$cd $LLVM_SRC/projects
	$git clone https://github.com/llvm-mirror/compiler-rt.git -b release_38 --single-branch --depth 1
	$mkdir build && cd build
	$cmake -DCMAKE_C_COMPILER="clang" -DCMAKE_C_FLAGS="-fPIC" -DCOMPILER_RT_BUILD_SANITIZERS=OFF ../compiler-rt
	$make
	$sudo make install
	$export BUILTIN_BUILD=$LLVM_SRC/projects/build

Download, compile and install musl-libc:
---------------------------------------
	// First, recompile Clang/LLVM for libc. Note: this should be a compiler option in production build
	$cd $LLVM_BUILD
	// set:
	// static bool IsLibc = true;
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp
	$cmake --build . --target LLVMX86CodeGen && sudo make install
	// Now deal with musl-libc
	$cd $BASE_DIR
	$export MUSL_BUILD=$BASE_DIR/musl-1.1.14
	$git clone https://github.com/lmrs2/musl-1.1.14.git
	$cd $MUSL_BUILD
	$LIBCC=/usr/local/lib/linux/libclang_rt.builtins-x86_64.a CFLAGS="-D_ZEROSTACK_=1 -fno-optimize-sibling-calls"  CC=clang ./configure --disable-static
	$make
	// You will see a lot of warning messages starting with "#define TAG_MUSL...": these are tags we will use for the CG version, just ignore them for now.
	// You will also see warnings about unsupported optimization flags. That's because clang and gcc are not in-sync with regards to flags.
	// A production build should look at each of them to fix the problem. 
	$sudo make install // this installs /usr/local/musl/bin/musl-clang which can be used to compile arbitrary programs
	// add the installation path to your path, eg
	$PATH=$PATH:/usr/local/musl/bin

Recompile Clang/LLVM for non-libc:
----------------------------------
	// recompile Clang/LLVM for non-libc. Note: this should be a compiler option in production build
	$cd $LLVM_BUILD
	// set:
	// static bool IsLibc = false;
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp
	$cmake --build . --target LLVMX86CodeGen && sudo make install

Example (function-based):
------------------------
	$cd $BASE_DIR
	$mkdir examples && cd examples
	$export EXAMPLES_DIR=`pwd`
	$vi function-based-main.c

Consider code below:

```c
#include <stdio.h>
#include <stdlib.h>

int foo(int a) {
	if ( a == 2) {
		printf("Winner\n");
		return 0;
	}
	if (a > 0) return -1;
	return -2;
}

int main(int argc, char * argv[]) {

	if ( !(argc > 1) ) {
		printf("Usage: %s <num>\n", argv[0]);
		return -1;
	}

	int v = atoi(argv[1]);
	int ret = foo(v);
	return ret;
}
```
	// Compile as
	$musl-clang function-based-main.c -o function-based-main
	$./function-based-main
	// every function now zeros registers and stack before the return instruction
	$objdump -d function-based-main
	# check linking dependencies
	$musl-ldd function-based-main


Recompile and install for stack-based implementation:
-----------------------------------------------------
	// Clang/LLVM
	$cd $LLVM_BUILD
	// set:
	// #define ZERO_EACH_FUNCTION 0
	// #define ZERO_EACH_FUNCTION_WITH_SIGNAL 0
	// #define ZERO_WITH_STACKPOINT	1 
	// #define ZERO_WITH_STACKPOINT_BULK_REG 1
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp
	$cmake --build . --target LLVMX86CodeGen && sudo make install

	// compiler_rt
	$cd $BUILTIN_BUILD
	$make clean && make && sudo make install

	// Clang/LLVM for libc
	$cd $LLVM_BUILD
	// set:
	// static bool IsLibc = true;
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp 
	$cmake --build . --target LLVMX86CodeGen && sudo make install

	// musl-libc
	$cd $MUSL_BUILD
	$make clean && make && sudo make install

	// Clang/LLVM for arbitrary (non-libc) programs
	$cd $LLVM_BUILD
	// set:
	// static bool IsLibc = false;
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp
	$cmake --build . --target LLVMX86CodeGen && sudo make install

Example (stack-based):
----------------------
	$cd $EXAMPLES_DIR
	$vi stack-based-main.c

Consider code below:

```c
#include <stdio.h>
#include <stdlib.h>

// only functions annotated with __sensitive will erase their stack/registers
#define __sensitive __attribute__((annotate("SENSITIVE")))

__sensitive int foo(int a) {
	if ( a == 2) {
		printf("Winner\n");
		return 0;
	}
	if (a > 0) return -1;
	return -2;
}

int main(int argc, char * argv[]) {

	if ( !(argc > 1) ) {
		printf("Usage: %s <num>\n", argv[0]);
		return -1;
	}

	int v = atoi(argv[1]);
	int ret = foo(v);
	return ret;
}
```
	// Compile as
	$musl-clang stack-based-main.c -o stack-based-main
	$./stack-based-main
	// functions marked as __sensitive zero registers and stack before the return instruction
	// all functions keep track of the stack usage - but do not erase it
	$objdump -d stack-based-main

Now let's try the callgraph-based implementation.

Install Gold linker (needed for LTO -- see http://llvm.org/docs/GoldPlugin.html):
--------------------------------------------------------------------------------
	cd $BASE_DIR
	// On Ubuntu Trusty (14.04), it is part of binutils package. If you followed the preliminary steps and installed binutils-dev, it should already be installed. Test it:
	$ld --version | grep -i gold
	// You can also try passing the -plugin option as:
	$ld -plugin
	// If it complains "unrecognized option '-plugin'", then you're using ld.bfd
	// It it says "-plugin: missing argument", then you're good
	// You may need to set the linker to the Gold linker thru a symbolic link, eg:
	$sudo rm /usr/bin/ld && sudo ln -s /usr/bin/ld.gold /usr/bin/ld 


Install python module dependencies:
--------------------------------
	$sudo apt-get install python-pip python-dev build-essential
	$sudo pip install --upgrade pip
	$sudo pip install future
	$git clone https://github.com/isislab/dispatch.git
	$cd dispatch
	$sudo python setup.py install
	// if the keystone installation fails, ignore it: we don't need it anyway
	$sudo pip install capstone

Download, compile and install additional passes:
-----------------------------------------------
	// this is based on Quala -- see https://github.com/sampsyo/quala
	$cd $BASE_DIR
	$git clone https://github.com/lmrs2/zerostack-callgraph.git
	$cd zerostack-callgraph/examples/fpointer
	$make && sudo make install
	// the above command installs
	//	1. CG-clang for compiling programs
	//	2. cg-compiler for compiling musl libc
	// 	3. create dummy empty file /tmp/metafile_pass

Recompile and install Clang/LLVM:
--------------------------------------------------------
	// Clang/LLVM
	$cd $LLVM_BUILD
	// set:
	// #define ZERO_WITH_STACKPOINT 0 
	// #define ZERO_WITH_STACKPOINT_BULK_REG 0
	// #define ZERO_WITH_CG 1 
	// #define ZERO_WITH_CG_BULK_REG 1 
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp 
	$cmake --build . --target LLVMX86CodeGen && sudo make install

Re-Configure, compile and install compiler_rt:
---------------------------------------------
	$cd $BUILTIN_BUILD
	$rm -rf *
	$cmake -DCMAKE_C_COMPILER="clang" -DCMAKE_RANLIB:FILEPATH=`llvm-config --bindir`/llvm-ranlib -DCMAKE_AR:FILEPATH=`llvm-config --bindir`/llvm-ar -DCMAKE_C_FLAGS="-fPIC -flto -Wimplicit-function-declaration -Werror" -DCOMPILER_RT_BUILD_SANITIZERS=OFF ../compiler-rt
	$make && sudo make install

Recompile Clang/LLVM for libc:
----------------------------------
	// recompile Clang/LLVM for libc. Note: this should be a compiler option in production build
	$cd $LLVM_BUILD
	// set:
	// static bool IsLibc = true;
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp 
	$cmake --build . --target LLVMX86CodeGen && sudo make install

Re-Configure, compile and install musl-libc:
--------------------------------------------
	$cd $MUSL_BUILD
	$make clean
	$cp Makefile.flto Makefile
	$LIBCC=/usr/local/lib/linux/libclang_rt.builtins-x86_64.a  CFLAGS="-D_ZEROSTACK_=1 -fno-optimize-sibling-calls -O3"  CC=cg-compiler ./configure --disable-static
	$make clean
	$make
	// as part of compilation, two files are generated:
	//	1. /tmp/metafile_pass -> call graph information generated by LLVM
	//	2. /tmp/metafile_pass.machine -> augmented call graph info with stack/register information generated by X86 backend: NAME(STACK_USED,LIST_REG_USED)
	$sudo make install
	// as part of the installation, two files are copied
	//	1. /tmp/metafile_pass -> /usr/local/musl/metafiles/musl-libc.mt
	//	2. /tmp/metafile_pass.machine -> /usr/local/musl/metafiles/musl-libc-machine.mt


Recompile Clang/LLVM for non-libc:
----------------------------------
	// recompile Clang/LLVM for non-libc. Note: this should be a compiler option in production build
	$cd $LLVM_BUILD
	// set:
	// static bool IsLibc = false;
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp
	$cmake --build . --target LLVMX86CodeGen && sudo make install

Example (callgraph-based):
-------------------------
	$cd $EXAMPLES_DIR
	$vi hash.c
```c
#include <stdio.h>
#include <stdlib.h>

#include "common.h"

__attr_hash_init void sha256_init(char *b, size_t len) {
  printf("sha256_init\n");
}

__attr_hash_init void sha512_init(char *b, size_t len) {
  printf("sha512_init\n");
}

```
	$vi callgraph-based-main.c
```c
#include <stdio.h>
#include <stdlib.h>
#include "common.h"

__attr_zerostack int main(int argc, char **argv)
{
   if (!(argc > 1)) {
     return -1;
   }
   hash_init func = NULL;
   int val = atoi(argv[1]);
   
   if (val == 4) func = &sha256_init;
   else if (val > 5) func = &sha512_init;

   if (func) (*func)(argv[1], val);

   return 0;
}

```
	$vi common.h
```c
#pragma once

// sensitive functions to zero stack/registers
#define __attr_zerostack __attribute__((annotate("SENSITIVE")))

// some annotations for callgraph
#define __attr_hash_init  __attribute__((type_annotate("hash_init")))
#define __attr_hash_init2  __attribute__((type_annotate("hash_init2")))

// this function pointer is annotated with __attr_hash_init
typedef __attr_hash_init void (*hash_init)(char *, size_t);

// those functions are annotated as __attr_hash_init
extern __attr_hash_init void sha256_init(char *b, size_t len);
extern __attr_hash_init void sha512_init(char *b, size_t len);
```
	$CG-clang -c -O3 -flto hash.c -o hash.o
	$CG-clang -c -O3 -flto callgraph-based-main.c -o callgraph-based-main.o
	$CG-clang -O3 -flto -o callgraph-based-main-unpatched callgraph-based-main.o hash.o
	// this generates:
	//	1. a file /tmp/metafile_pass.machine which contains callgraph info with registers and stack amount used
	// 	2. a binary with code to zero stack and registers. However which registers to zero and what amount of stack to zero is done
	// separately. This is because LLVM backend does not support module passes, so it only sees functions one at a time and cannot
	// determine the call graph to compute the stack usage. The metafiles you saw earlier have this information instead; and the following python script
	// patches the binary
	$python $BASE_DIR/zerostack-callgraph/examples/fpointer/patchme.py --help
	$python $BASE_DIR/zerostack-callgraph/examples/fpointer/patchme.py --inobject=callgraph-based-main-unpatched --outobject=callgraph-based-main-patched --inmetafiles=/usr/local/musl/metafiles/musl-libc-machine.mt,/tmp/metafile_pass.machine --libc=musl --platform=x86_64 --signal-stack-use=4096 --bulk-register-zeroing
	$./callgraph-based-main-patched
	$objdump -d callgraph-based-main-patched

