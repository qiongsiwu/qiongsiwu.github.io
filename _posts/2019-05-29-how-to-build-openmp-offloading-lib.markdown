---
layout: post
title:  "Building LLVM OpenMP Target Offloading Library"
date:   2019-05-29 10:55:19 -0400
categories: llvm openmp
---
Building llvm has been quite troublesome during the days of svn. 
Now the entire llvm repository has been moved to github, 
life is much easier. Building the OpenMP target offloading library
is no longer something impossible to do correctly on the first 
try. Jonas Hahnfeld wrote [a very detailed instruction][hahnjo-omp-offloading]
on how the OpenMP target offloading library should be build. This post
builds on top of that to show how to build llvm OpenMP target offloading 
library using the [new github repo][llvm-github].

# Prerequisites
First, all the following tools are necessary before attempting the build. 
```bash
gcc   # use version 7 or up. This may not be necessary but it won't hurt. 
g++
git
cmake
ninja
``` 
Additionally, if you encounter issues about missing prerequisites, 
consult the llvm [getting started][llvm-getting-started] page. 

# Building Clang and Clang++
1. Obtain the source code 
```bash
git clone https://github.com/llvm/llvm-project.git
```
Or if you prefer ssh, 
```bash 
git clone git@github.com:llvm/llvm-project.git
```

2. Create a build directory inside `llvm-project`. 
```bash
mkdir build
cd build
```

3. Configure the build with cmake. 
```bash
cmake -G Ninja \
	-DLLVM_ENABLE_PROJECTS=‘clang;openmp’ \
	-DLIBOMPTARGET_NVPTX_COMPUTE_CAPABILITIES=61 \
	-DCMAKE_INSTALL_PREFIX=$(pwd)/../install/ \
	-DCMAKE_BUILD_TYPE=Release ../llvm
```
	There are a few things to note about this configuration.
	* The LLVM projects enabled are necessary to build OpenMP target offloading
libraries. You can add more if you need them. 
	* The compute capabilities is set to 61 in this case because I am running
a GTX 1080 Ti. Change it to the compute capabilities your graphics card 
supports. You can use a comma separated list to specify multiple compute 
capabilities at once. Make sure this is specified when building the compiler.
Otherwise, the linker will have difficulties when building OpenMP
applications. 
	* The install folder is set to `llvm-project/install`. 
	* The build type is set to release. If you want to have debug infomation, 
you can set the type to `Debug` or `RelWithDebInfo`.

4. Build and install Clang/LLVM. Make sure you are in the `build` directory
and invoke the following command. 
```bash
ninja install
``` 

5. Go get some coffee or have a walk. Depending on your machine, 
building clang and LLVM can take forever. Ninja compiles using multiple 
threads already but it can still take 20+ minutes to build on a work station
with a 14 core CPU with hyper-threading turned on. 

# Building OpenMP Target Offloading Libraries
Now you are back from coffee, and llvm is installed. We can proceed to 
build the OpenMP libraries. The important thing about this step is that
we will need the clang/clang++ built by the previous steps. Don't use 
the `gcc` that comes with your system. 

To build the OpenMP libraries, 
1. In the `llvm-project` directory, 
```bash
mkdir buildomp
cd buildomp
```  
2. Configure the project. Set the C and C++ compilers to 
clang and clang++.  
```bash
cmake -G Ninja \
	-DCMAKE_C_COMPILER=$(pwd)/../install/bin/clang \
	-DCMAKE_CXX_COMPILER=$(pwd)/../install/bin/clang++ \
	-DLIBOMPTARGET_NVPTX_COMPUTE_CAPABILITIES=61 \
	-DCMAKE_INSTALL_PREFIX=$(pwd)/../install \
	-DCMAKE_BUILD_TYPE=Release \
	../openmp
```
	Note that we still need to specify the compute capabilities here. 
3. Build the libraries.
```bash
ninja install
``` 

# Use the Libraries
To use the new compilers and libraries, you can add them to your `PATH`
and `LD_LIBRARY_PATH` respectively. 
```bash
# In the llvm-project directory
export PATH=$(pwd)/install/bin:$PATH
export LD_LIBRARY_PATH=$(pwd)/install/lib:$LD_LIBRARY_PATH
```

Now you can compile some C code with OpenMP offloading to see if everything 
works. 

```bash
clang -fopenmp -fopenmp-targets=nvptx64 -Xopenmp-target -march=sm_61 -O3 
omp_hello.c
```

[hahnjo-omp-offloading]: https://www.hahnjo.de/blog/2018/10/08/clang-7.0-openmp-offloading-nvidia.html
[llvm-github]: https://github.com/llvm/llvm-project
[llvm-getting-started]: https://llvm.org/docs/GettingStarted.html
