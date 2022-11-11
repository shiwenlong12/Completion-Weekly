
# 实验目的  
一：实现多任务并存和任务切换，前面的批处理系统虽然说也是有多个应用程序在我们的系统当中运行，但是每个时刻它只能有一个程序在里面，从这一章开始我们的多个程序可以同时存在于我们的系统当中，还有任务之间可以切换。  
二：实现一一个时钟周期为时间片的时间片轮转调度，就是每一次发生时钟中断的时候，我们都会做一个任务切换。  


# 实验过程
## 一：多道程序的编译、打包和载入内存
对chapter2中的脚本进行简单的修改，即可让多道程序同时存在于内核程序的代码段中，（见每段代码开头的注释）。
当chapter==3的时候，会app_id += 1,这样它每一次会往上叠加一个step，这样的话它每一个程序的入口点就隔了一个step，而如果说没有app_id += 1的话，如在chapter2里面app_id始终都是0，它就会在同一个地方为起点，这就是本章和上一章的区别。  
然后就是不同的app也会被载入到代码段中不同的内存位置，不同应用的代码已经在我们的内核程序的数据段当中，并且我们都记录了一个它的位置，我们现在要把它载入到代码段当中不同的位置，这样的话它们才能够并存。  
get_base_i(app_id:usize) -> usize的功能就是说我们把第app_id号应用的代码段被我们存在了什么地方，我们要去什么地方去寻找它的一个地址。load_app()的主要功能是遍历每一个应用app，获取了app的基地址，然后存放app的内存一开始置0，然后就是把属于app_start[i]的整个程序做成切片存在src里面，然后dst就是我们的目标位置，就是把它复制到base_i为起点，src.len()为长度，就是把原程序从数据段移动到对应的我们放置存储的位置，然后我们还需要为每个app分配各自的KernelStack和USER_STACK,在批处理系统中KernelStack和USER_STACK都是唯一的一个，因为可以复用，但是在多道程序系统中因为我们要同时存在多个，所以我们不能复用了，所以我们用一个数组为每一个任务都分配了一个内核栈和一个用户栈。最后，对各app的TrapContext的初始化也随之变化，我们为了建立一个进程，我们需要把对应的TrapContext推入到内核栈当中来模仿一个起始的状态，这个时候因为它不同的栈不一样了，所以在这里也有了一个数组。  
这里我们可以看到，跟chapter2相比的话，它其实也就是考虑到了载入的位置不一样，其它的基本上是完全一样的。

## 二：任务的保存与切换
### 任务保存
与前一个的chapter的方法类似，内核实例化了数据结构TaskManager，该数据结构包含了一个用来存储所有任务状态信息的全局数组以及其他整体性的描述信息，而描述任务状态信息的数据结构为TaskControlBlock，其中包含了任务的状态TaskStatus和其运行信息TaskContext，由此，当我们只要有任务的app_id，就可以访问到其任意信息。  
在这里我们能看到它是怎么管理不同的app的，首先我们知道我们的系统当中会有多个任务同时存在，任务存在的时候会有一个TaskControlBlock结构，这个结构就可以代表我们的任务这个时候的一个状态，包括这个任务是就绪态、等待态还是退出态还是启动态，包括任务的上下文等等，这些都存放在我们的任务控制块里面。其实我们有了一个任务控制块我们就可以定位到这个任务的所有的状态，所以我们的操作系统里面必须要能存放一个任务控制块的一个数组，每个任务控制块都代表了不同的应用。  
然后，我们再定义一个结构TaskManager，它里面就会包含一个总体描述的信息，在这里就是有多少个tasks，然后利用inner:UPsafeCell<TaskManagerInner>进行一种封装，这个封装的好处是：如果没有这个封装，当我们去访问这个全局变量的时候，我们经常要使用UPsafeCell，这对我们来说是非常不友好的，我们用rust的原因就是因为rust安全，用UPsafeCell的时候顾名思义是不安全的，所以为了维护它的安全性，我们把这种全局性的一个结构用一个智能指针把它抱起来，在这里封装一层就是为了去修改这个全局变量。TaskManagerInner包括任务控制块的数组，与一个current_task标识了我们此时正在运行的是哪一个task。  
然后就是我们的一个实例化，我们有了TASK_MANAGER这个数据结构之后，我们需要在系统里面定义一个全局的实例，然后用它来做整个系统里面所有task的一个管理。它的初始化的过程是tasks表示一个数组，在里面首先有一个zero_init（）对每一个进程的上下文进行了定义，然后是task_status，在刚开始构建的时候，TaskControlBlock本来相当于控制块的值，此时还没有任务进来，当有任务进来的时候，他就会占用一个槽，这个时候我们才会把它的状态改变，所以在刚定义的时候task_status一定是UnInit这个状态。然后我们对tasks里面的取num_app，也就是有num_app个任务进来了，进来了之后，我们会给它的任务上下文和任务状态初始化，刚进来的任务肯定是处于Ready就绪态的，然后我们就可以返回了，这样我们就做好了一个初始化的工作。  
在这里，若存在app_id号的app，则TASK_MANAGER[app_id]的Task_Context完成初始化，状态设为Ready；否则，Task_Context为Task_Context::zero.init(),状态为Uninit。
### 任务切换
任务切换是本章比较核心的部分，切换在外层有一个pub fn __switch(current_task_cx_ptr: *mut TaskContext,next_task_cx_ptr:*const TaskContext)函数，它有两个参数，current_task_cx_ptr表示当前上下文的指针，它在自己的内核栈的栈顶上面；next_task_cx_ptr表示要切换的任务上下文的指针，在对方的内核栈栈顶上面。如果我们调用了这个切换函数__switch，实现为汇编形式，通过汇编代码可以看出，__switch()其实就做了一个保存与恢复，将寄存器中的内容保存在自身的TrapContext中，再把对方的TrapContext写入寄存器中，即完成了切换。而任何一个任务的TrapContext均可以通过TASK_MANAGER找到，因此，问题就变成寻找合适的app_id了。剩下的便都是策略和细节问题了。  
### 选择合适的任务进行切换
这一部分代码主要在os/src/task/mod.rs中。这一部分会告诉我们如何选择合适的任务进行切换，里面有run_first_task()函数，我们刚刚初始化的时候，我们的计算机处于内核态里面，我们总需要选第一个task来执行，这个函数就是去运行第一个task；然后run_next_task()就是某一个程序执行完了或者説因为一些原因中断了报错了，那我们就一个去找下一个任务来执行；mark_current_suspended就是说我们要实现时间片轮转，当某一个进程正在运行的时候，我们先让它停下来不要跑了，把它的状态从Rinning改成Ready；mark_current_excited()就是正在执行的程序现在要退出了，退出需要做一些处理，就是把状态从Running改为Exited；suspend_current_and_run_next()功能就是先让当前的任务停下来然后跑下一个任务；exit_current_and_run_next()功能就是先让当前的任务退出然后跑下一个任务。  

接下来我们看看这几个函数的底层的实现，这几个函数都包裹在impl TaskManager下，即它们都是对TASK_MANAGER的实例进行了一个方法的调用，所以我们需要看数据结构里面是怎么定义这几个函数的。run_first_task()目的功能是选择app_id为0的task作为第一个执行的task，函数前三行实际上可以看成一个固定的公式，我们如何去修改每一个task控制块里面的值，后面想要修改进程控制块里面的内容的时候，都可以用这个方法。想要运行第一个task，根据__switch函数接口只要执行了__switch()我们就可以转移到它。我们先要找到他的地址在哪里，通过let next_task_cx_ptr = &task0.task.cx as *const TaskContext;找到了他的上下文的位置并且把它转换成指针，然后inner没用了我们需要把它drop掉，let mut _unused = TAskContext::zero_init();在上下文切换的时候有一个当前任务的上下文和一个将要切换的任务的上下文，实际上当前任务的上下文是什么状态其实在第一次执行的时候没有什么意义，因为我们不会再切换到这个状态了，所以我们随便设了一个；执行完了unsafe{__switch(&mut _unused as *mut TaskContext,next_task_cx_ptr);}之后，即进行切换之后，我们就会切换到了第一个任务，第一个任务是tasks[0]意味着我们跑到app_id=0的地方去执行了。  
接下来实现的就是使当前的任务状态改为Ready，fn mark_current_suspended(&self){let mut inner = self.inner.exclusive_access();let current = inner.current_task;inner.tasks[current].task_status = TaskStatus::Ready;},显然它只是把状态变成了Ready，它如果不执行一个切换的话，那他还会跑，所以我们后面还要跟一个切换。  
find_next_task(&self) -> Option<usize>功能是寻找下一个切换的task的app_id，基本方法为循环查找第一个状态为Ready的task。（current+1..current+self.num_app+1).map(|id| id%self.num_app).find(|id|inner.tasks[id].task_status==TaskStatus::Ready)，这个函数返回的是一个Option<app_id>，我们知道有了app_id我们就可以进行切换了。  
run_next_task(&self)功能是切换到下一个task，包含修改该task的状态，修改调度器的current_task，以及做切换操作。具体实现是首先找到下一个task的app_id记为next，let Some(next) = self.find_next_task()，我们找到next之后，我们把inner.tasks[next].task.status变成Running，然后inner.current_task = next将next作为当前任务，完成之后我们需要做切换，需要把两个TrapContext作为参数，然后调用_switch()，所以需要找到这两个参数，之后执行_switch()函数进行切换。至此，我们的主体的函数已经实现了。

## 增加系统调用yield()和exit()
pub fn sys_yeild() -> isize叫做调度，也就是说现在的进程放弃CPU了，它很简单，它只需要执行suspended_current_and_run_next();他就可以放弃CPU了。然后退出函数pub fn sys_exit(exit_code:i32)需要执行exit_current_and_run_next();。现在进程已经可以通过调用sys_yeild()和sys_exit()主动退出CPU了。  

## 支持时钟中断以及在时钟中断中增加task切换
### 时钟中断支持
在main.rs的pub fn rust_main()函数中调用使能函数trap::enable_timer_interrupt();实际上使能时钟中断就是修改了一个寄存器的位而已，pub fn enable_timer_interrupt(){unsafe{sie::set_timmer();}}；我们的时钟中断实际上是来至于操作系统时钟，操作系统时钟在我们的这个架构里面实际上是通过timer函数timer::set_next_trigger();可以设置下一次触发中断的时间，然后设计时间set_next_trigger()的实现是pub fn set_next_trigger(){set_timer(get_time() + CLOCK_FREQ/TICKS_PER_SEC);}，CLOCK_FREQ/TICKS_PER_SEC可以看成一个时间间隔，所以这个的含义就是在当前时间的基础上加上一个时间间隔，到了那个时间的时候我们就触发一次时钟中断，所以就实现了经过这个时间间隔来触发一次中断；get_time()和get_time_us()就是获取时间的函数，不过get_time_us()是以微秒作为基本单位的。然后我们在trap/mod.rs的pub fn trap_handler(cx:&mut TrapContext) -> &mut TrapContext里面也要做一些修改，就是Trap::Interrupt(Interrupt::SupervisorTimer) =>{set_next_tigger();suspended_current_and_run_next();}；SupervisorTimer就是说时钟中断的时候我们怎么做，我们需要设置下一个时钟，利用set_next_tigger();把时钟设在下一个时间间隔的位置；然后需要执行切换，把切换函数suspended_current_and_run_next();放进来就可以了。实际上只需要在以前的框架上做一些小的修改就可以了，至此，时钟中断和切换task就完成了。

# 编程练习
## 题目：获取任务信息  
我们已经实现了多个任务分时轮流运行，我们希望引入一个新的系统调用，fn sys_task_info(ti: *mut TaskInfo) -> isize，我们希望吧TaskInfo放进我们传进来的参数里面，struct TaskInfo {status: TaskStatus,syscall_times: [u32; MAX_SYSCALL_NUM],time: usize}，syscall_times是表示系统调用次数的数组，例如我们有0号系统调用、1号系统调用、2号系统调用一直到包括最大的系统调用，所有的系统调用都有一个编号，u32表示每一号系统调用我们一共调用了多少次，第几号系统调用的次数就是数组中的第几个数字u32；time表示从这个任务第一次被调度能够执行开始，到当前为止一共过去了多长时间，也就是説就算这个task在Ready状态等CPU的时间也算上；所以要得到time很简单，只需要知道第一次被调度是什么时候，然后每次查看time的时候用get_time()得到当前时间，然后减去开始时间就可以了。
## 完成步骤
### 1、在TaskControlBlock中增加相应的元素来记录需要的值
首先我们需要知道每一个进程都有自己的time，也都有自己的syscall_times，那么最好的方式就是把它们记录在控制块TaskControlBlock中，所以我们在TaskControlBlock中增加两个项，一个是syscall_times，一个是start_time，我们向TCB增加了内容，那么在TCB进行各种初始化和赋值的时候我们也需要进行修改,因为在之前初始化和赋值的时候没有考虑这两个变量。  
### 2、为所有初始化和赋值TCB的函数增加对应的初始化和赋值  
在本次实验中，只有在pub static ref TASK_MANAGER: TaskManager = {.....let mut tasks = [TaskControlBlock｛task_cx:TaskContext::zero_init(),
task_status:TaskStatus::UnInit,syscall_times:[0,MAX_SYSCALL_NUM],start_time:0,
｝；MAX_APP_NUM];.....}有初始化操作，我们把syscall_times和start_time加上。
### 3、在task第一次被调度的时候为其start_time赋值
在这里我们用了一个小trick，用start_time本身来做一个flag，当调度到某task时发现其start_time为0，则说明该task第一次被调度，将start_time设计为当前时间，否则说明该task已经被调度过，不修改其start_time。
所以在fn run_next_task(&self){
    if let Some(next) = self.find_next_task(){
        ...
        if inner.tasks[next].start_time == 0{
            inner.tasks[next].start_time = get_time_us();
        }
        .....
    }
}
我们在next第一次我们准备让它执行的时候，我们加了一个判断，如果start_time是0的话，就把当前时间给它。
### 4、增加pub方法使得其他模块也能够获取到当前task的start_time和taskstatus，从而获取需要的信息
在之前，我们只是加在了TaskManager里面，然后实际上我们需要给程序提供一个公共的接口，这样的话让我们的框架更加清晰，所以我们增加了两个公共函数去调用里面的start_time和获取它的状态status,因为我们的info里面还要必须有一个状态。
impl TaskManager{
    .....
    fn get_current_TaskControlBlock_start_time(&self) -> usize{
        let inner = self.inner.exclusive_access();
        let current = inner.current_task;
        inner.tasks[current].start_time
    }
}
pub fn get_current_start_time() -> usize{
    TASK_MANAGER.get_current_TaskControlBlock_start_time();
}
### 5、增加syscall_times的记录和处理
我们首先在TaskManager中增加了一个函数fn add_syscall_times(&self,syscall_id:usize)功能是增加某一个sysacll_id的系统调用计数，这个很简单，就是我们只要调用了这个函数，那么某一个syscall的计数就会加一，然后我们同样给它提供两个对外的接口，一个是add_syscall_times(syscall_id:u32)，一个是get_syscall_times()->[u32,MAX_SYSCALL_NUM],因为我们最后获取info的时候需要get一下。我们怎么时候应该去调用add_syscall_times增加syscall的计数呢，当然是syscall/mod.rs里面当某一个系统调用马上就要执行的时候，我们先做一步add,这样的话就能够记录该进程的任何一个syscall次数。
pub fn syscall(syscall_id:usize,args:[usize;3]) -> isize{
    add_syscall_times(syscall_id);
    .....
}
### 6、实现系统调用sys_task_info
pub fn sys_task_info(ti:*mut TaskInfo) -> isize{
    unsafe{
        *ti = TaskInfo{
            status:get_current_status(),
            syscall_times:get_syscall_times,
            time:(get_time_us()-get_current_start_time())/1000,
        };
    }
    0
}
至此，本实验的主体框架和编程练习已经完成。

