# ollvm/armariris/hikari port to llvm10 in 2020


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



