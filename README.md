# Completion-Weekly
操作系统毕业设计周报

第六周报告
  
  本周工作总结：
    一：引入了进程的概念，不再通过硬编码来执行，而是通过加载elf数据的形式来执行。在task/processor.rs中增加了更新系统调用次数、得到系统调用次数、得到进程运行时间、设置优先级、申请内存、释放内存等函数。
    二：在syscall/process.rs中实现了spawn函数，新建子进程，使其执行目标程序。spawn函数基本上就是fork函数和exec函数结合起来的，主要区别就是不像fork函数一样复制父进程的地址空间。
    三：在task/manager.rs中实现了stride调度算法，计算方式是 P.stride = BigStride / P.priority,每次执行一个任务需要inner.pass += stride,然后再调度时选择pass最小的执行。因为priority当了分母，所以优先级越大stride就越小，进程就越先被调用。

第五周报告

本周工作总结：

一：经过和陈林峰同学的讨论，lab1的代码没有问题，是实验的配置问题，开始的问题需要将Cargo.toml的最后三行注释掉，任然有问题是因为为了能够完成实验环境的配置将rust-toolchain里的时间从04-11改为另了09-30，将时间改回04-11，就能够完成make test3,但是会上传失败Error: The process '/home/runner/.cargo/bin/cargo' failed with exit code 101。

二：在lab2中重写了sys_get_time和sys_task_info的代码，它们失效的原因是因为引入了虚拟内存的机制，ts和ti不是真正的物理地址而是虚拟地址，因此只需要添加一个地址转换就可以解决这两个函数失效的问题，通过get_refmut来实现虚拟地址到物理地址的转换，地址转换实际上就是查表。


三：增加了mmap和munmap两个系统调用于申请和释放内存，代码框架已经实现了MemorySet负责管理一个应用所获得的所有内存，因此只需要向MemorySet里新加入一个MapArea表示一段连续的内存，mmap需要检查起始地址对齐问题，对应地址是否有效，插入页表项后是否成功；munmap需要注意释放区域是否有效，释放后检查是否还在页表项内。

四：完成了lab2的实验要求，地址为https://github.com/LearningOS/lab2-os4-shiwenlong12

五：学习了第四章进程及进程管理。

下周工作目标：

一：完成lab3和lab4的实验。

第四周报告

本周工作总结：

一：在loadeer.rs中内核同时加载多个应用，在task子模块中进行任务管理，即协作式调度和抢占式调度；实现了多个任务分时轮流运行。

二：实现sys_task_info功能来获取当前任务的信息，在task子模块中task.rs的TaskControlBlock中添加syscall_times得到系统调用次数，添加start_time得到开始时间，在task子模块中mod.rs中先调用get_time_us得到开始时间，添加get_current_task得到当前任务的信息和increase_syscall_times设置系统调用的次数；并对外提供接口。在syscall子模块的process.rs中得到当前任务的状态、系统调用次数和总共运行时间。

下周工作目标：

一：继续完成上周未完成的任务。

第三周报告

本周工作总结：

一：在陈林峰同学的帮助下，修改rust_toolchain版本，解决了环境配置出错的问题。

二：修改了/.github/workflows/classroom.yml中的toolchain，得到最新的工具链，解决了自动评分的问题。

三：实现并分析了应用程序基本执行的源代码，地址为：https://github.com/LearningOS/lab0-0-setup-env-run-os1-shiwenlong12

四：实现并分析了批处理系统的源代码，地址为：https://github.com/LearningOS/lab0-1-run-os2-shiwenlong12

五：实现了部分多道程序与分时多任务的源代码，地址为：https://github.com/LearningOS/lab1-os3-shiwenlong12

下周工作目标：

一：继续完成多道程序与分时多任务的源代码。

二：完成地址空间的实现源代码。


第二周报告

本周工作总结：

一：完成了rustlings练习，GitHub地址为：https://github.com/LearningOS/learn_rust_rustlings-shiwenlong12

二：学习了B站里的Rust中文社区群线上学习室——通过Rustlings学习Rust,地址为 https://space.bilibili.com/24917186

三：进行了实验环境的配置

下周工作目标：

一：完成前三个kernel blabs 实践。
