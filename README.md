# Completion-Weekly
操作系统毕业设计周报

第五周报告

本周工作总结：

一：经过和陈林峰同学的讨论，lab1的代码没有问题，是实验的配置问题，开始的问题需要将Cargo.toml的最后三行注释掉，任然有问题是因为为了能够完成实验环境的配置将rust-toolchain里的时间从04-11改为另了09-30，将时间改回04-11，就能够完成make test3,但是会上传失败Error: The process '/home/runner/.cargo/bin/cargo' failed with exit code 101。



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
