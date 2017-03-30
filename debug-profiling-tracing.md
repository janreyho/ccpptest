# 调试性能分析及跟踪

- 调试 (debugging) - 寻找程序运行时行为异常的原因并解决的过程;
- 性能分析 (profiling) - 分析程序运行时行为以提供对关键指标 (执行速度/资源占用/...) 的统计性结论;
- 跟踪 (tracing) - 在时域上记录程序运行行为，为后续调试或性能分析提供第一手数据;

所有的调试、性能分析和跟踪工具都依赖于某种 逻辑注入(instrumentation) 机制，注入可以是静态的 (编译期发生) 也可以是动态的 (运行期发生)

## C/C++调试工具

### 实现原理

- 核心功能:断点硬件断点
  - x86 体系下使用调试寄存器 DR0 ̃7
  - DR0 ̃3 - 4 个独立的硬件断点线性地址
  - DR6(DR4) - 断点触发状态
  - DR7(DR5) - 断点控制
- 软件断点
  - x86 体系下使用 INT3 (0xcc) 机器指令
  - 自行抛出 SIGTRAP 信号模拟可移植软件断点
- 虚拟机解释器
  - 逐条解释而不是直接批量执行目标程序代码
- Linux 平台上用户态调试的基础设施
  - ptrace 系统调用

### ptrace

ptrace提供了一种使父进程得以监视和控制其它进程的方式，它还能够改变子进程中的寄存器和内核映像，因而可以实现断点调试和系统调用的跟踪。

使用ptrace，你可以在用户层拦截和修改系统调用(sys call)

[ptrace运行原理及使用详解](http://blog.csdn.net/edonlii/article/details/8717029)

### GDB通用调试器

- 基于 ptrace 系统调用实现
- 同时支持硬件和软件断点 - hbreak/break
- 7.x 以上版本支持  向调试 功能
  - 通过在执行每条机器指令前保存其修改的寄存器/内存地址内容实现，开销很大但很有用


- 例子:

  独立调试:	gdb --args <exec> [<arg1> [...]]

  分析:		coredump - gdb <exec> <core>

  调试运行中进程:	gdb <exec> <pid>	

#### coredump

操作系统就会把程序当掉 时的内存内容 dump 出来（现在通常是写在一个叫 core 的 file 里面），让 我们或是 debugger 做为参考。这个动作就叫作 core dump。

- 小程序产生 core dump

  ```
  #include <stdio.h>

  int main()
  {
      int *null_ptr = NULL;
      *null_ptr = 10;            //对空指针指向的内存区域写,会发生段错误
      return 0;
  }

  ```

  ```
  #编译执行
  guohailin@guohailin:~$ ./a.out
  Segmentation fault (core dumped)
  guohailin@guohailin:~$ ls      #多出下面一个 core 文件
  -rw-------  1 guohailin guohailin 200704 10月 22 11:35 a.out.core.22070    
  ```

#### 使用 gdb 调试 Core 文件

产生了 core 文件，我们该如何使用该 Core 文件进行调试呢？Linux 中可以使用 GDB 来调试 core 文件，步骤如下：

- 首先，使用 gcc 编译源文件，加上 `-g` 以增加调试信息；
- 按照上面打开 core dump 以使程序异常终止时能生成 core 文件；
- 运行程序，当core dump 之后，使用命令 `gdb program core` 来查看 core 文件，其中 program 为可执行程序名，core 为生成的 core 文件名。

### Valgrind - 动态调试分析工具框架

Valgrind是用于构建动态分析工具的探测框架。它包括一个工具集，每个工具执行某种类型的调试、分析或类似的任务，以帮助完善你的程序。Valgrind的架构是模块化的，所以可以容易地创建新的工具而又不会扰乱现有的结构。

- 本质上是一个指令级解释器/虚拟机框架，故 不可能 用来调试运

行中进程:

- 有用的调试用插件:

1. memcheck：内存相关错误检查器
2. massif： 运行时堆栈使用情况分析器
3. helgrind：线程错误检查器
4. DRD：另一类线程错误检查器
5. sgcheck (原 ptrcheck)：栈/全局数组越界错误检查器
6. Callgrind：它主要用来检查程序中函数调用过程中出现的问题。
7. **Extension。**可以利用core提供的功能，自己编写特定的内存调试工具。
8. **Cachegrind**。它主要用来检查程序中缓存使用出现的问题。

#### memcheck

检查启动进程及其派生的所有进程中的内存错误:

```bash
valgrind --tool=memcheck --leak-check=full --leak-resolution=high --track-origins=yes --trace-children=yes --log-file=result.log <exec>
```
查看 memcheck 插件特有选项的帮助:

````bash
valgrind --tool=memcheck --help
````

[如何使用Valgrind memcheck工具进行C/C++的内存泄漏检测](https://www.oschina.net/translate/valgrind-memcheck)

[应用 Valgrind 发现 Linux 程序的内存问题](https://www.ibm.com/developerworks/cn/linux/l-cn-valgrind/)

#### massif

第二种类型的内存泄漏，就是，长期闲置的内存堆积。这部分内存虽然你还保存了地址，想要释放的时候还是能释放。关键就是你忘了释放。。。杯具啊。这种问题memcheck就不给力了。这就引出了第二个内存工具valgrind –tool=massif。

 统计进程整个生命期内的堆栈使用情况:

```bash
valgrind --tool=massif --stacks=yes <exec>
ms_print massif.*
```

ms_print 输出统计图中:

: 表示普通采样快照，其只采集对应时刻的堆栈使用量
@ 表示详细采样快照，除堆栈使用量外其还会采集调用栈
\# 表示整个生命期中的使用量峰值

````
    MB
95.44^                                                         #
     |                                                         #:
     |                                                        :#:
     |                                                       ::#:@
     |                                                      :::#:@:
     |                                                     ::::#:@::
     |                                                     ::::#:@::
     |                                                    :::::#:@:::
     |                                                   ::::::#:@::::
     |                                                  :@:::::#:@:::::
     |                                                  :@:::::#:@::::@
     |                                                 ::@:::::#:@::::@:
     |                                                :@:@:::::#:@::::@::
     |                                                :@:@:::::#:@::::@::
     |                                               @:@:@:::::#:@::::@::@
     |                                              :@:@:@:::::#:@::::@::@:
     |                                             ::@:@:@:::::#:@::::@::@::
     |                                            :::@:@:@:::::#:@::::@::@::
     |                                            :::@:@:@:::::#:@::::@::@::@
     |                                           @:::@:@:@:::::#:@::::@::@::@:
   0 +----------------------------------------------------------------------->Mi
     0                                                                   2.127
````

[valgrind Massif](http://blog.csdn.net/unix21/article/details/9330571)

#### helgrind/DRD/sgcheck

- helgrind：检查 POSIX 线程 API 误用 / 锁顺序不一致 / 竟态条件等错误:

```
valgrind --tool=helgrind <exec>
```

- DRD：检查 POSIX 线程 API 误用 / 竟态条件 / 锁竞争错误，并跟踪所有mutex 操作:

````
valgrind --tool=drd --trace-mutex=yes <exec>
````

- sgcheck：检查栈/全局数组访问越界错误:

```
valgrind --tool=exp-sgcheck <exec>
```

Valgrind 版本较低时应将 exp-sgcheck 换为 exp-ptrcheck