# ollvm/armariris/hikari port to llvm10 in 2020

现在代码保护技术很多是在 llvm 上实现的，例如 ollvm 和 hikari ，作者给出的实现是将源码混杂在 llvm中，这样做非常不优雅。近来越来越多安全工作者都开始接触和研究基于 llvm的代码保护，工欲善其事必先利其器，在编译、运行均是本机的环境下，不会出问题，因此本文介绍的是，如何优雅地在NDK中加载pass。

安卓开发者使用混淆技术来保护native代码时，一般有两种选择：

第一个选择是获得git上 ollvm或 hikari的代码，编译后，替换掉NDK中原先的toolchain。

这是最不优雅的方式，因为维护起来很麻烦，因为需要编译整个llvm工程，并且对NDK有侵入性，无法保证修改前和修改后NDK的功能不发生变化。

第二个选择是，编译llvm工程，替换掉NDK中原先的toolchain，并且在相同环境下，移植 ollvm或 hikari为独立的plugin，（移植方案我的github里有写https://github.com/LeadroyaL/llvm-pass-tutorial）用编译为插件的形式，动态加载插件。

相比第一个方案，极大降低维护的代价，只编译一个pass即可，但仍然对NDK有侵入性。

这两种方案的共同特点是：都需要编译整个llvm项目，初次部署时要消耗大量的时间和资源，另外在选择llvm版本时，也会纠结适配性的问题（虽然通常不会出现适配问题）

笔者曾经使用的是第二种方案，经过研究，本文提出第三种方案，使用NDK中的环境编译pass并加载pass，优雅程度上来看，有以下的特点：

最最重要的，不需要编译llvm项目，节省巨大的时间和资源消耗；
其次，不修改原先的NDK运行环境，和原生的NDK是最像的，没有侵入性；
再次，上下文均和NDK完全一致，不需要担心符号问题，不需要额外安装软件和环境，有NDK的环境就足矣；


使用NDK的环境编译一个pass
众所周知，编译Pass时需要使用 llvm的环境，由于NDK中的 llvm环境是破损的，所以开发者一般自己编译一份 llvm环境出来，替换掉NDK中的llvm环境，包括我本人之前也是这样处理的，这样做的原因是NDK中的 llvm是破损的，因为NDK来自AOSP编译好的 toolchain，而AOSP在制作 toolchain的过程中是移除了部分文件的。

上文提到，本文的方案是不需要亲自编译llvm的，因此就需要使用NDK中的破损的llvm环境来编译一个pass。

根据对 https://android.googlesource.com/toolchain/llvm_android/ 的阅读和调试，NDK中的llvm缺失的是一部分binary文件、全部静态链接库文件、全部头文件，采用的是静态连接的方式，它的clang是较为独立的文件（它会依赖libc++，因此称为较为独立）。

平时编译Pass时，需要使用cmake并且导入各种cmake相关的环境，通常写如下的配置文件

https://github.com/abenkhadra/llvm-pass-tutorial/blob/master/CMakeLists.txt

```c
cmake_minimum_required(VERSION 3.4)
# we need LLVM_HOME in order not automatically set LLVM_DIR
if(NOT DEFINED ENV{LLVM_HOME})
    message(FATAL_ERROR "$LLVM_HOME is not defined")
else ()
    set(ENV{LLVM_DIR} $ENV{LLVM_HOME}/lib/cmake/llvm)
endif()
find_package(LLVM REQUIRED CONFIG)
add_definitions(${LLVM_DEFINITIONS})
include_directories(${LLVM_INCLUDE_DIRS})
link_directories(${LLVM_LIBRARY_DIRS})
add_subdirectory(skeleton)  # Use your pass name here.
```

幸运的是，NDK中的 lib/cmake/llvm 还在，里面的cmake文件都是原汁原味的的。

不幸的是，由于AOSP在编译toolchain时设置了 defines['LLVM_LIBDIR_SUFFIX'] = '64' ，导致find_package的路径应该是 lib64/cmake/llvm ，需要稍加修改。

之后进行

mkdir b;cd b;cmake ..
 
mkdir b;cd b;cmake ..





# llvm 错误相关

## 一、因为 LLVM_ENABLE_ABI_BREAKING_CHECKS 引起的符号缺失

常见表现：

这两个符号 Not Found

__ZN4llvm23EnableABIBreakingChecksE

__ZN4llvm24DisableABIBreakingChecksE

触发场景：

使用NDK的clang加载自己编译的、开启LLVM_ENABLE_ABI_BREAKING_CHECKS的 Pass（即使用关闭LLVM_ENABLE_ABI_BREAKING_CHECKS的/path/bin/clang 加载开启了LLVM_ENABLE_ABI_BREAKING_CHECKS的 pass）

原因：

开启LLVM_ENABLE_ABI_BREAKING_CHECKS后，符号为__ZN4llvm23EnableABIBreakingChecksE，关闭LLVM_ENABLE_ABI_BREAKING_CHECKS后，符号为__ZN4llvm24DisableABIBreakingChecksE。二者就差几个字符，很容易眼花被看错，让 clang 和 pass保持一致就行。
```c
namespace llvm {
  #if LLVM_ENABLE_ABI_BREAKING_CHECKS
  extern int EnableABIBreakingChecks;
  __attribute__((weak, visibility ("hidden"))) int *VerifyEnableABIBreakingChecks = &EnableABIBreakingChecks;
  #else
  extern int DisableABIBreakingChecks;
  __attribute__((weak, visibility ("hidden"))) int *VerifyDisableABIBreakingChecks = &DisableABIBreakingChecks;
  #endif
}


namespace llvm {
  #if LLVM_ENABLE_ABI_BREAKING_CHECKS
  extern int EnableABIBreakingChecks;
  __attribute__((weak, visibility ("hidden"))) int *VerifyEnableABIBreakingChecks = &EnableABIBreakingChecks;
  #else
  extern int DisableABIBreakingChecks;
  __attribute__((weak, visibility ("hidden"))) int *VerifyDisableABIBreakingChecks = &DisableABIBreakingChecks;
  #endif
}
```
解决方案：
编译时主动设置或者清空该宏定义。

## 二、因为libc++和stdlibc++不一致引起的符号缺失

常见表现：

以下符号 Not Found，其实并不是这一个符号缺失，而是缺失了一大堆符号，这个是第一个。

undefined symbol: 

_ZNK4llvm12FunctionPass17createPrinterPassERNS_11raw_ostreamERKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE’

触发场景：

使用 NDK 自带的 clang，加载一个 gcc 编译出来的 pass，会报这个错。

原因：

/path/bin/clang在生成过程中，如果使用gcc构建整个llvm，生成的 bin/clang会使用 stdlibc++那一套符号，如果使用clang构建整个llvm，生成的 bin/clang会使用 libc++那一套符号。虽然它们来自同一份源码，但二者对 c++的函数处理是不一致的，同一个函数会被解析为如下两种

_ZNK4llvm12FunctionPass17createPrinterPassERNS_11raw_ostreamERKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE

_ZNK4llvm12FunctionPass17createPrinterPassERNS_11raw_ostreamERKNSt3__112basic_stringIcNS3_11char_traitsIcEENS3_9allocatorIcEEEE

解决方案：

编译 pass 时使用和 /path/bin/clang 一致的 c++实现，stdlibc++和 libc++都试一遍。

## 三、因为是否使用cxx11引起的符号缺失

常见表现：

以下符号 Not Found，其实并不是这一个符号缺失，而是缺失了一大堆符号，这个是第一个。

undefined symbol: _ZNK4llvm12FunctionPass17createPrinterPassERNS_11raw_ostreamERKSs’

触发场景：

编译/path/bin/clang 时使用低版本的 g++（小于5），编译pass 时使用了高版本的 g++（大于等于 5），常见于clang 和 pass 是从两台设备中抠出来的。

原因：其实和第一种是同一种情况，g++在高版本默认使用 cxx11 的特性，高版本和低版本的符号不一致。

解决方案：

编译 pass 时使用和 /path/bin/clang 一致的 c++实现，要么使用 cxx11 的特性，要么别使用 cxx11 的特性。

## 四、因为clang没有该导出符引起的符号缺失

常见表现：

所有符号全都找不到

__ZN4llvm12FunctionPass17assignPassManagerERNS_7PMStackENS_15PassManagerTypeE

__ZN4llvm23EnableABIBreakingChecksE

__ZN4llvm24DisableABIBreakingChecksE

触发场景：

使用 macOS 上系统自带的 clang 去加载任何的 Pass。使用 macOS 上的 NDK里的 clang 去加载任何的 Pass。

➜ /tmp nm -gU -demangle /usr/bin/clang
0000000100000000 T __mh_execute_header
0000000100000f77 T _main
0000000100002008 S _shim_marker

➜ /tmp nm -gU -demangle /usr/bin/clang
0000000100000000 T __mh_execute_header
0000000100000f77 T _main
0000000100002008 S _shim_marker

原因：

因为没有很多符号，加载 shared library 时直接失败，苹果自带的 clang 是自己的体系，而 NDK 里的纯属因为 strip 的 bug。https://issuetracker.google.com/issues/143160164

解决方案：

无法解决，请不要使用这两个 clang 去加载 pass。请自己编译一个clang出来用。

## 五、pass在某些情况下被加载了但是没有被执行

常见表现：什么报错都没有，就是不执行。

可能原因：已知一个是静态连接导致的，其实挺少见的。



