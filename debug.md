## 目标

- 开发、测试时调试
- 运行时搜集必要的运行环境、统计信息
  - OS, env, stack
  - 运行时间长度, API callcnt, GPU使用状态
  - 版本c/s

## Python

### faulthandler  打印backtrace

原理: 当python程序发生异常（`SIGSEGV`, `SIGFPE`, `SIGABRT`, `SIGBUS`, and `SIGILL`）时，通过注册的c代码，自动打印程序堆栈，可输出到文件或者stdout/stderr. // 没有segmentfault??

方式：

1. 程序启动时 设置环境变量

   ```shell
   PYTHONFAULTHANDLER=1 or
   -X faulthandle
   ```

2. 代码运行时，enable

   ```python
   import faulthandler
   faulthandler.enable(file=2, all_threads=True)
   # faulthandler.register(signum, file=sys.stderr, all_threads=True, chain=False)
   ```

限制：

- python3.3开始支持，python2.7 官方不支持，可以下载patch
  - e.g., <https://github.com/vstinner/faulthandler> 
- 只能打印ASCII字符，字符串限制在500个
- 堆栈只能显示filename, funName, lineNime
- 最多100个线程和100个堆栈

参考：

- https://docs.python.org/3/library/faulthandler.html



### python core dump

1. 产生core dump
   - host
   - docker
2. 调试core dump
3. 

## C 代码控制

### catchsegv

原理： linux utility 首先设置signal handler，然后启动app，打印到stderr

例子：

限制：

- linux平台



### 加载libSegFault.so

```shell
# running
export SEGFAULT_SIGNALS="all"       # "all" signals
export SEGFAULT_SIGNALS="bus abrt"  # SIGBUS and SIGABRT
LD_PRELOAD=/lib/libSegFault.so program -o hai

or 
# compiling
gcc -g1 -lSegFault -Wl,--no-as-needed -o program program.cc
```

### GCC built-in

GCC also has two builtins that can assist you, but which may or may not be implemented fully on your architecture, and those are `__builtin_frame_address` and `__builtin_return_address`. Both of which want an immediate integer level (by immediate, I mean it can't be a variable). If `__builtin_frame_address` for a given level is non-zero

### StackWalker

支持平台：Linux/x86, Linux/AMD-64, Linux/Power, Linux/Power-64, BlueGene/L, and BlueGene/P

Ref: <https://dyninst.org/stackwalker>

https://dyninst.org/sites/default/files/downloads/dyninst/8.2.1/DyninstAPI-8.2.1.tgz 

### breakpad

google开源的C/S方式 crash-reporting 产生-管理-基本分析的工具，保护Client lib, dumper util, processor

<https://chromium.googlesource.com/breakpad/breakpad>



### libunwind

google开源的perftools，x86, ARM 参考：<http://www.nongnu.org/libunwind/> 

The API additionally provides the means to manipulate the preserved (callee-saved) state of each call-frame and to resume execution at any point in the call-chain (non-local goto). The API supports both local (same-process) and remote (across-process) operation. As such, the API is useful in a number of applications. Some examples include:

- exception handling

  The libunwind API makes it trivial to implement the stack-manipulation aspects of exception handling.

- debuggers

  The libunwind API makes it trivial for debuggers to generate the call-chain (backtrace) of the threads in a running program.

- introspection

  It is often useful for a running thread to determine its call-chain. For example, this is useful to display error messages (to show how the error came about) and for performance monitoring/analysis.

- efficient setjmp()

  With libunwind, it is possible to implement an extremely efficient version of setjmp(). Effectively, the only context that needs to be saved consists of the stack-pointer(s).

## 注册signal handler

### 例子

- backtrace and backtrace_symbols_fd are not async-signal-safe. you should not use these function in signal handler
- 如果malloc出错，可能无法工作

```c
// gcc -g -rdynamic ./test.c -o test
// use rdynamic to print out backtraces using the backtrace()/backtrace_symbols() of Glibc.
//Without -rdynamic, you cannot get function names
// -rdynamic may increase the size of the binary relatively significantly in some cases
//by default signal handler is called with the same stack and SIGSEGV is thrown twice. To protect you need register an independent stack for the signal handler.
#include <stdio.h>
#include <execinfo.h>
#include <signal.h>
#include <stdlib.h>
#include <unistd.h>

#ifdef STACK_OVERFLOW
// avoid stackoverflow!!!!!!!
static char stack_body[64*1024];
static stack_t sigseg_stack;
#endif
static struct sigaction sigseg_handler;

void handler(int sig) {
  void *array[10];
  size_t size;

  // get void*'s for all entries on the stack
  size = backtrace(array, 10);//最好用stack

  fprintf(stderr, "Error: signal %d:\n", sig);
  backtrace_symbols_fd(array, size, STDERR_FILENO); //没有用malloc
  signal(sig, SIG_DFL);
  _exit();// exit() not safely from a signal handler. Use _exit() or _Exit(), or abort()
   // or kill(gettpid(), sig)
  // and better to free(array)， 
}

void baz() {
 int *foo = (int*)-1; // make a bad pointer
  printf("%d\n", *foo);       // causes segfault
}

void bar() { baz(); }
void foo() { bar(); }

int main(int argc, char **argv) {
#ifdef STACK_OVERFLOW
  sigseg_stack.ss_sp = stack_body;
  sigseg_stack.ss_flags = SS_ONSTACK;
  sigseg_stack.ss_size = sizeof(stack_body);
  assert(!sigaltstack(&sigseg_stack, nullptr));
  sigseg_handler.sa_flags = SA_ONSTACK;
#else
  sigseg_handler.sa_flags = SA_RESTART;  
#endif
  sigseg_handler.sa_handler = &handler;
  assert(!sigaction(SIGSEGV, &sigseg_handler, nullptr));

  //signal(SIGSEGV, handler);   // install our handler
  foo(); // this will call foo, bar, and baz.  baz segfaults.
}
```



## C++ 

C++中多态函数名问题，mangles all symbols. --> demangling 

<http://charette.no-ip.com:81/programming/2010-01-25_Backtrace/> 

```shell
$ g++ -rdynamic 2010-01-25_Backtrace.c && ./a.out 
0: ./a.out(_Z16displayBacktracev+0x26) [0x4009da]
1: ./a.out(_Z3barv+0x9) [0x400a4a]
2: ./a.out(_Z3foov+0x9) [0x400a5a]
3: ./a.out(main+0x9) [0x400a65]
4: /lib/libc.so.6(__libc_start_main+0xfd) [0x7f4ecbea1abd]
5: ./a.out [0x4008f9]
```

开源工具：backward.hpp 一个头文件, backward.cpp

- https://github.com/bombela/backward-cpp  MIT license, from Google

- 依赖libfd libdw libdwarf libelf

- ```shell
  apt-get install binutils-dev libdw-dev libdwarf-dev
  #define BACKWARD_HAS_BFD 1 // before include backward.hpp
  #define BACKWARD_HAS_DW 1
  g++/clang++ -lbfd -ldl -g3 -rdynamic -DBACKWARD_HAS_DW=1 -DBACKWARD_HAS_BFD=1 -DBACKWARD_HAS_DWARF=1
  ```

- 



## 参考

- <https://pythonspeed.com/articles/python-c-extension-crashes/>
- <https://stackoverflow.com/questions/77005/how-to-automatically-generate-a-stacktrace-when-my-program-crashes> 

