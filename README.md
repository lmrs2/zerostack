zerostack: Zeroing stack and registers of sensitive functions
=============================================================
This project contains Clang/LLVM patches/passes to zero the stack and registers of sensitive functions. There are 
3 solutions implemented, as per the paper "What you get is what you C: Controlling side effects in mainstream C compilers", 
available at [TODO]. The code is a reference implementation, and is not supposed to be production ready. Nevertheless, 
this should be ebough for anyone who wants to test it.

The setup below was tested on Ubuntu trust 14.04.5 LTS x86_64. I suggest you install this as a VM before reading further.

Pre-requesites:
---------------
	1. Install a VM running Ubuntu trust 14.04.5 LTS x86_64. Allocate 32GB of disk.
	2. $sudo apt-get install git cmake g++ binutils-dev
	3. $sudo apt-get install gcc-multilib g++-multilib
	4. $export BASE_DIR=/whereever/you/want
	5. $cd $BASE_DIR

Download, compile and install Clang/LLVM (function-based implementation):
-----------------------------------------------------------
	$git clone https://github.com/lmrs2/llvm.git -b zerostack_38 --single-branch --depth 1 
	$cd llvm
	$cd tools
	$git clone https://github.com/lmrs2/clang.git -b zerostack_38 --single-branch --depth 1 
	$cd ..
	$mkdir build
	$cd build
	$cmake -DLLVM_BINUTILS_INCDIR=/usr/include -DBUILD_SHARED_LIBS=ON -DCMAKE_BUILD_TYPE=Debug -DLLVM_TARGETS_TO_BUILD="X86" ../
	$cmake --build .
	$sudo make install
	$export LLVM_SRC=`llvm-config --src-root`
	$export LLVM_BUILD=$LLVM_SRC/build

Download, compile and install compiler_rt:
-----------------------------------------
	$cd $LLVM_SRC/projects
	$git clone https://github.com/llvm-mirror/compiler-rt.git -b release_38 --single-branch --depth 1
	$mkdir build
	$cd build
	$cmake -DCMAKE_C_COMPILER="clang" -DCMAKE_C_FLAGS="-fPIC" -DCOMPILER_RT_BUILD_SANITIZERS=OFF ../compiler-rt
	$make
	$sudo make install
	$export BUILTIN_BUILD=$LLVM_SRC/projects/build

Download, compile and install musl-libc:
---------------------------------------
	// First, recompile Clang/LLVM for libc. Note: this should be a compiler option in production build
	$cd $LLVM_BUILD
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp // set static bool IsLibc = true;
	$cmake --build . --target LLVMX86CodeGen && sudo make install
	// Now deal with musl-libc
	$cd $BASE_DIR
	$export MUSL_BUILD=$BASE_DIR/musl-1.1.14
	$git clone https://github.com/lmrs2/musl-1.1.14.git
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
	// recompile Clang/LLVM for libc. Note: this should be a compiler option in production build
	$cd $LLVM_BUILD
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp // set static bool IsLibc = false;
	$cmake --build . --target LLVMX86CodeGen && sudo make install

Example (function-based):
------------------------
	TODO
	+ objdump



Recompile and install for stack-based implementation:
-----------------------------------------------------
	// Clang/LLVM
	$cd $LLVM_BUILD
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp // set ZERO_EACH_FUNCTION to 0 and ZERO_WITH_STACKPOINT to 1 and ZERO_WITH_STACKPOINT_BULK_REG to 1
	$cmake --build . --target LLVMX86CodeGen && sudo make install

	// compiler_rt
	$cd $BUILTIN_BUILD
	$make clean && make && sudo make install

	// Clang/LLVM for libc
	$cd $LLVM_BUILD
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp // set static bool IsLibc = true;
	$cmake --build . --target LLVMX86CodeGen && sudo make install

	// musl-libc
	$cd $MUSL_BUILD
	$make clean && make && sudo make install

	// Clang/LLVM for arbitrary (non-libc) programs
	$cd $LLVM_BUILD
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp // set static bool IsLibc = false;
	$cmake --build . --target LLVMX86CodeGen && sudo make install

Example (function-based):
------------------------
	TODO
	+ objdump

Now let's try the callgraph-based implementation/

Install Gold linker (needed for LTO -- see http://llvm.org/docs/GoldPlugin.html):
--------------------------------------------------------------------------------
	cd $BASE_DIR

	$sudo apt-get install texinfo bison flex

	// On Ubuntu Trusty (14.04), it is part of binutils package. If you followed the preliminary steps and installed binutils-dev, it should already be installed. Test it:
	$ld --version | grep -i gold
	// You can also try passing the -plugin option as:
	$ld -plugin
	// If it complains "unrecognized option '-plugin'", then you're using ld.bfd
	// It it says "-plugin: missing argument", then you're good
	// You may need to set the linker to the Gold linker thru a symbolic link, eg:
	$sudo ln -s /usr/bin/ld.gold /use/bin/ld 


Install python module dependencies:
--------------------------------
	$sudo apt-get install python-pip python-dev build-essential
	$sudo pip install --upgrade pip
	$sudo pip install future
	$git clone https://github.com/isislab/dispatch.git
	$cd dispatch
	$sudo python setup.py install
	// if the capstone installation fails, then
	$sudo pip install capstone
	TODO: test

Download, compile and install additional passes:
-----------------------------------------------
	// this is based on Quala -- see https://github.com/sampsyo/quala
	$cd $BASE_DIR
	$git clone https://github.com/lmrs2/zerostack-callgraph.git
	$cd zerostack-callgraph/examples/fpointer
	$make
	$sudo make install
	// the above command installs
	//	1. CG-clang for compiling programs
	//	2. cg-compiler for compiling musl libc
	/ 	3. create dummy empty file /tmp/metafile_pass

Recompile and install Clang/LLVM:
--------------------------------------------------------
	// Clang/LLVM
	$cd $LLVM_BUILD
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp // set ZERO_WITH_CG to 1 and ZERO_WITH_CG_BULK_REG to 1 and ZERO_WITH_STACKPOINT to 0 and ZERO_WITH_STACKPOINT_BULK_REG to 0
	$cmake --build . --target LLVMX86CodeGen && sudo make install

Re-Configure, compile and install compiler_rt:
---------------------------------------------
	$cd $BUILTIN_BUILD
	$rm -rf *
	$cmake -DCMAKE_C_COMPILER="clang" -DCMAKE_RANLIB:FILEPATH=`llvm-config --bindir`/llvm-ranlib -DCMAKE_AR:FILEPATH=`llvm-config --bindir`/llvm-ar -DCMAKE_C_FLAGS="-fPIC -flto -Wimplicit-function-declaration -Werror" -DCOMPILER_RT_BUILD_SANITIZERS=OFF ../compiler-rt
	$make
	$sudo make install

Recompile Clang/LLVM for libc:
----------------------------------
	// recompile Clang/LLVM for libc. Note: this should be a compiler option in production build
	$cd $LLVM_BUILD
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp // set static bool IsLibc = true;
	$cmake --build . --target LLVMX86CodeGen && sudo make install

Re-Configure, compile and install musl-libc:
--------------------------------------------
	$cd $MUSL_BUILD
	$cp Makefile.flto Makefile
	$make clean
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
	// recompile Clang/LLVM for libc. Note: this should be a compiler option in production build
	$cd $LLVM_BUILD
	$vi ../lib/Target/X86/X86ZeroStackPass.cpp // set static bool IsLibc = true;
	$cmake --build . --target LLVMX86CodeGen && sudo make install

Example:
-------
	$CG-clang blabla -o main
	// this generates a binary with code to zero stack and registers. However which registers to zero and what amount of stack to zero is done
	// separately. This is because LLVM backend does not suppot module pass, so it only sees function one at a time and cannot
	// compute the stack usage using call graph. The metafiles you saw earlier have this information instead; and the following python script
	// patches the binary
	$python patchme.py --inobject=main --outobject=main-patched --inmetafiles=/usr/local/musl/metafiles/musl-libc-machine.mt,/tmp/metafile_pass.machine --libc=musl --platform=x86_64 --signal-stack-use=4096 --bulk-register-zeroing
	$./main-patched
