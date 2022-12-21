chapter4:地址空间  
/*  
chapter4与之前相比，难度大了很多，在我们前面的章节当中，我们访问内存的方式都是通过一个物理地址的方式，我们直接给出物理地址，然后计算机会到对应物理地址的地方去访问对应的内存单元，但是到了本章地址空间，我们开启了一个映射机制，也就是说我们给出的是一个虚拟地址，然后虚拟地址会经过一系列硬件机制的映射，把这个虚拟地址转化为一个对应的物理地址。这样的话，我们实际上是使用虚拟地址去访问，通过硬件机制去进行一个地址转化，这个硬件叫MMU，通过这个转化之后，我们的计算机会到对应的物理地址的位置去访问。  
*/  


# 实验逻辑和基本设计
## 实验逻辑
/*  
1、我们的操作系统可以同时存在多个不同的任务，我们可以把它们称为进程。  
1、页表就是给我们的进程的地址映射做服务的，虚拟地址到物理地址就是依靠物理地址来完成的，  
3、页表就是提供虚拟页到物理页转化的数据结构，我们进程分配虚拟页之后，是一定要在进程当中找一个对应的物理页来进行一个映射的，一个进程产生了一个新的虚拟页，我们需要一个物理页来对它进行绑定，因此我们需要有探测内存中的空闲物理页帧的机制。  
3、我们的页表可以看成是一个一个数组，然后这个数组里面存的结构就叫做页表项，然后我们地址映射靠的就是解析页表项，然后这就要求我们能够知道这个页表项的物理地址的话，就需要能够直接访问到它，就需要这个接口。  
*/  
1、实验目的为：进程创建时，为每一个进程分配一张页表（实际上还有更高级的抽象：地址空间和逻辑段，但是都是以页表结构为核心），能够在页表中实现为进程分配新的虚拟页以及释放任意虚拟页的功能。  
2、进程的页表均在内核空间中，为了能够给新进程分配页表，需要提供分配页表的策略。  
3、为了每个进程都能够自由的分配和回收自身任意虚拟页，需要有探测内存中的空闲物理页帧以及回收任意物理页帧的机制，并且必须有访问和修改随机页表项的接口。

## 基本设计
1、rcore tutorial采用双页表的设计，即内核使用单独的地址空间，不同的用户进程在用户态时使用各自的用户地址空间，这也就意味着地址空间的切换发生在特权级切换时。  
/*  
进程之间的切换页表的时候实际上是这样一个过程：先从我们的a进程进入到我们的内核态下，这个时候会发生一次地址空间的切换，然后在这个时候我们在内核空间里面把b进程地址空间相关的上下文放到TrapContext里面去，然后我们在进行一个恢复，就从内核的地址空间又切换到了b的地址空间。  
*/  

2、内核空间采用对等映射（恒等映射）的方式，即内核的任意PageNum号虚拟页对应的物理页框号也是PageNum，这样的话，在内核中访问物理页时便不再会受到MMU的影响。内核页表在操作系统初始化的时候便一并完成对等映射的初始化了。  
/*  
比如说1086号的虚拟页对应的也是1086的物理页帧，这样做的好处是：在内核里，我们经常要去访问不同的地址空间，然后我们可以读它的页表，我们知道它的虚拟页对应的物理页在哪里，但是我们的映射机制一旦打开就不能再关闭了，那我们的内核想要非常方便的去访问一个物理页该怎么办呢，我们就要使用对等映射的方式，也就是说，实际上我们想要访问1086号的物理页我们只需要内核的1086号虚拟页就可以了。  
*/  

3、每个用户进程在初始化时，同步为其分配一个物理页作为其页表的根节点，也就是最高级的一个页表，然后，我们在给它增加页表项的时候在进行进一步的处理，此后，所有的操作均在该页表上实现。  
/*  
我们进程页表进来的时候当然只有一个新的，然后分配它的虚拟页，然后它的页表才能逐渐的增加页表项。  
*/  

4、chapter4的核心内容在于对页表的支持，这里的页表被封装成了一个PageTable的数据结构，这个结构实际上控制了所有被该页表占用的页框，需要实现的接口包括：  
    1）创建一个新的页表  
    2）进程分配或回收一个虚拟页时，需要去修改对应的页表项，因此，页表必须能够随机访问和修改页表项。  
    3）为了满足1)、2）的要求，必须能探测和分配空闲物理页帧以及回收物理页帧。  
/*  
随机访问和修改页表项:我们是用虚拟页号去访问页表项的，这是个三级页表结构，它如果想要去访问对应的页表项的话，他可能需要经过三次映射才能够找到，但是我们在设计这个页表的时候，我们实际实现的时候是通过三级页表去找到页表项，但是我们必须做好一个封装，我们这里面的细节给隐藏起来，也就是说我们必须设计直接通过一个虚拟页号就可以找到那个页表项，我们必须找到这个接口，这样的话就好像直接经过一次映射就找到了，三次映射的细节被我们封装在了接口里面。  
也就是说，这个接口一旦实现了，我们就可以直接通过虚拟页去找到对应的那个物理页所对应的那个页表项，而不用每次都去做这个三次映射处理。  
*/  
## 注意
本次实验采用三级页表结构，所以任意一个虚拟页的访问和处理必须经过三次查表：先通过一级页号去访问根页表对应的页表项，获取对应的二级页表的地址，在通过二级页号去访问对应二级页表中的页表项，从中取出三级页表的地址，在通过三级页号去访问对应页表项，从中获取地址对应的物理页帧。


# 实验过程
## 1、构建页表相关的数据结构和方法
### 建立页与地址之间的转换关系
    // os/src/mm/address.rs

    #[derive(Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
    pub struct PhysAddr(pub usize);

    #[derive(Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
    pub struct VirtAddr(pub usize);

    #[derive(Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
    pub struct PhysPageNum(pub usize);

    #[derive(Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
    pub struct VirtPageNum(pub usize);
/*  
页表是让我们虚拟页和物理页之间做转换，但是我们实际上去访问数据的时候是通过虚拟地址和物理地址，虚拟地址是我们给出来的，物理地址是实际上去内存当中访问的那个地址，所以我们需要一些基本的结构，并且做了一些封装。其实这些结构都是简单的包裹了一个usize结构，主要是为了更好的增加类型方法，来进行类型装换。  
*/  

    // os/src/mm/address.rs
    impl PhysAddr {
        pub fn page_offset(&self) -> usize { self.0 & (PAGE_SIZE - 1) }
    }
    impl From<PhysAddr> for PhysPageNum {
        fn from(v: PhysAddr) -> Self {
            assert_eq!(v.page_offset(), 0);
            v.floor()
        }
    }
    impl From<PhysPageNum> for PhysAddr {
        fn from(v: PhysPageNum) -> Self { Self(v.0 << PAGE_SIZE_BITS) }
    }
/*  
我们这里是用来from triat里面的from来对它们进行处理，，通过这个处理之后我们就可以usize到物理页，到虚拟页、到物理地址、到虚拟地址都可以通过from函数轻易的实现了。  
*/  

这里增加了各种结构与usize的from关系，通过这样的方式我们可以用较为简单和通用的写法去获取结构包裹的usize值。   

/*  
比方说，我们想要一个虚拟页vp，我们有了一个usize，我们应该怎么做呢，我们通过这样一个from调用：let vp = VirtPageNum::from(p);，然后就可以封装好一个VirtPageNum，如果我们想要获取里面的，我们vp.into就可以了。我们当然也可以使用vp.0。  
*/  
    let p:usize = 1086;  
    let vp = VirtPageNum::from(p);  
    let p2:usize = vp.into;  

/*  
之前，我们封装的是usize与虚拟页、usize与虚拟地址、usize与物理页、usize与物理地址之间的关系，但是实际上，我们更重要的关系是虚拟地址跟虚拟页之间的关系，物理地址跟物理页之间的关系。我们可以想一下，一个地址如果是分成一个页一个页的，那它一定有自己所在的那个页，我们的虚拟地址所在的虚拟页，那怎么转换成虚拟页，就是调用floor(&self) -> VirtPageNum函数，ceil(&self) -> VirtPageNum函数的功能是：比方说某一个地址在第五页上面，那ceil（）函数会到它后面那个页里面，到第六页，它永远是取到了地址的后面一个页，但是有一种特殊情况，就是当它是某一页的第一个位置，也就是边界的那个位置的话，ceil()函数和floor()函数是一样的，都表示当前的页。  
我们这样设计的原因是，到后面我们会用到很多区间设计，就是一个页的区间以及一个地址的区间都是一个左闭右开的结构，这样的话更方便它的一个处理。  
总之floor()和ceil()都是把虚拟地址转化成虚拟页。  

page_offset(&self) -> usize功能是：页中有一个业内偏移page_offset，会提供它的偏移；然后aligned(&self) -> bool是一个判断函数，表示它的偏移量是不是刚好为0，也就是说判断是不是我们刚才说的在某一个页的首元素；然后就是一个转化。  
物理页和虚拟页也是一样。  

还有一个我们前面说了，就是页号是分成一级页号，二级页号，三级页号，然后虚拟页实际上被分成了27位，然后前9位是一级页号，中间9位是二级页号，最后9位是三级页号，然后我们希望能够很简单的把它给分别获取到，这样的话有利于我们后面去访问三个页表，这样的话在虚拟页的页号结构VirtPageNum里面就有一个indexes(&self) -> [usize;3]方法，然后返回一个三维数组，然后然后里面分别存放一级页号，二级页号，三级页号。  
*/  


我们后面经常会虚拟页空间还是连续的，为了方便的表示这一段连续的虚拟页并方便对它的处理，我们做了一些新的定义：
    pub trait StepByOne{
        fn step(&mut self);
    }
    impl StepByOne for VirtPageNum{
        fn step(&mut self){
            self.0 += 1;
        }
    }
为VirtPageNum定义了自增函数step()，并加入到StepByOne这一trait中。  

/*  
首先，我们定义了一个新的trait叫StepByOne，它实际上就是自增1的意思。这样的话，我们希望有一个结构能够表示一串虚拟页的空间的，我们首先设计了一个叫SimpleRange的结构，它是一个区间的结构，r表示区间的上界，r表示开区间的下界，也就是说实际上这个区间是取不到这个r的，这点我们需要注意，后面的设计也是这样。  
get_end()函数也是一样，取不到r，只能取到r前面一个。  
*/  

定义了结构体SimpleRange来表示一段区间：  
    #[derive(Copy, Clone)]
    /// a simple range structure for type T
    pub struct SimpleRange<T>
    where
        T: StepByOne + Copy + PartialEq + PartialOrd + Debug,
    {
        l: T,
        r: T,
    }

构建了创建SimpleRange的函数，并提供了可以访问它边界的接口。  
    impl<T> SimpleRange<T>
    where
        T: StepByOne + Copy + PartialEq + PartialOrd + Debug,
    {
        pub fn new(start: T, end: T) -> Self {
            assert!(start <= end, "start {:?} > end {:?}!", start, end);
            Self { l: start, r: end }
        }
        pub fn get_start(&self) -> T {
            self.l
        }
        pub fn get_end(&self) -> T {
            self.r
        }
    }

创建了SimpleRange的迭代器。
    pub struct SimpleRangeIterator<T>
    where
        T: StepByOne + Copy + PartialEq + PartialOrd + Debug,
    {
        current: T,
        end: T,
    }
    impl<T> SimpleRangeIterator<T>
    where
        T: StepByOne + Copy + PartialEq + PartialOrd + Debug,
    {
        pub fn new(l: T, r: T) -> Self {
            Self { current: l, end: r }
        }
    }
    impl<T> Iterator for SimpleRangeIterator<T>
    where
        T: StepByOne + Copy + PartialEq + PartialOrd + Debug,
    {
        type Item = T;
        fn next(&mut self) -> Option<Self::Item> {
            if self.current == self.end {
                None
            } else {
                let t = self.current;
                self.current.step();
                Some(t)
            }
        }
    }

/*  
我们使用这个区间主要就是为了方便使用它的迭代器，我们有了一个区间之后，比方说我们想要访问地20页到第200页之间所有的虚拟页，那我们其实只需要创建一个迭代器，然后就可以使用迭代器的方法遍历访问里面所有的虚拟页了。  
这些封装的接口了解一下就行了，我们只需要知道迭代器有一个new()的方法，传入的参数是下界与上界，然后我们就可以使用这个迭代器对区间里面所有的元素进行遍历。  
*/  

最后，我们对以上结构进行封装，就可以获取VirtPageNum类型的Range类型了。
    /// a simple range structure for virtual page number
    //一种简单的虚拟页码范围结构
    pub type VPNRange = SimpleRange<VirtPageNum>;  
此后，我们便可以使用VPNRange来对一段虚拟页区间做操作了。  

/*  
我们用VPNRange来表示虚拟页相关的区间。这个区间结构对我们来说在后面实验设计的时候还挺好用的，我们会在后面具体举一个例子来说明怎么使用这个区间结构。  
*/  


### 构建页表项
页表项结构是硬件决定的，在本实验中，页表项是64位二进制，其中[53:10]这44位是物理页号，最低的8位[7:0]则是标志位，它们的含义如下：
    仅当 V(Valid) 位为 1 时，页表项才是合法的；  
    R/W/X 分别控制索引到这个页表项的对应虚拟页面是否允许读/写/取指；  
    U 控制索引到这个页表项的对应虚拟页面是否在 CPU 处于 U 特权级的情况下是否被允许访问；  
    G 我们不理会；  
    A(Accessed) 记录自从页表项上的这一位被清零之后，页表项的对应虚拟页面是否被访问过；  
    D(Dirty) 则记录自从页表项上的这一位被清零之后，页表项的对应虚拟页表是否被修改过。  

页表存放在内存中，我们想要设置某个虚拟页的页表项，只需要在对应的内存位置装入我们写好的页表项就可以了，页表项的结构为：
    #[derive(Copy, Clone)]
    #[repr(C)]
    /// page table entry structure
    pub struct PageTableEntry {
        pub bits: usize,
    }

/*  
这样的话，我们只需要与刚才一样，我们用一个usize来代表一个页表项，当然了，页表项的存储很简单，主要还是对页表项的解析。我们的页表项里面存的就2个，一个是物理页号，一个是标记，因此我们需要通过bits把它们反解析出来，而且给了物理页号与标志位也要能够把bits给构建出来。  
*/  

一个usize正好对应64位二进制，因此可以代表一个页表项，我们的重点是构建页表项的处理函数：  
    new(ppn: PhysPageNum, flags: PTEFlags) -> Self、  
    empty() -> Self、  
    ppn(&self) -> PhysPageNum、  
    flags(&self) -> PTEFlags、  
    is_valid(&self) -> bool、  
    readable(&self) -> bool、  
    writable(&self) -> bool、  
    executable(&self) -> bool。  
/*  
is_valid()表示这个页表项是不是有效的，什么时候会有效，曾经装过这个虚拟页，并且我们没有让这个虚拟页失效，也就是说这个页表项是可以用的，曾经定义过这个页表项，页表项对应的虚拟页时可以用的，这样V标志位为1，页表项就叫有效。  
*/  

此后，我们只需要查表就可以找到对应的物理页帧，当目标物理页帧存放的是页表的时候，我们希望以pte指针的形式来访问它；当目标页帧存放程序数据的时候，我们则希望能够按指定的类型来访问它，因此，为了方便起见，我们给PhysPageNum提供了如下方法。  
    //src/mm/address.rs
    impl PhysPageNum {
        pub fn get_pte_array(&self) -> &'static mut [PageTableEntry] {
            let pa: PhysAddr = (*self).into();
            unsafe { core::slice::from_raw_parts_mut(pa.0 as *mut PageTableEntry, 512) }
        }
        pub fn get_bytes_array(&self) -> &'static mut [u8] {
            let pa: PhysAddr = (*self).into();
            unsafe { core::slice::from_raw_parts_mut(pa.0 as *mut u8, 4096) }
        }
        pub fn get_mut<T>(&self) -> &'static mut T {
            let pa: PhysAddr = (*self).into();
            unsafe { (pa.0 as *mut T).as_mut().unwrap() }
        }
    }

获取的指针都是 *mut类型，这意味着我们可以通过指针来修改对应的数据。  
/*  
现在，我们想要访问或者构建一个页表项的虚拟页的话，三级页表结构分成了一级页表、二级页表、三级页表，然后我们获得了一个页号之后，我们有一个根页表，我们从根页表里面找到了一个页表项，我们本来就有根页表的起始的物理地址，然后我们根据一级页号知道了它应该是位于最顶级页表的第多少项，然后我们就需要去访问这个页表项，之后，我们从这个页表项里面获取了对应的二级页表的起始地址，然后我们又根据他的二级页表去找。这样的话有些麻烦，所以我们希望有一个快速的根据物理地址去找到对应的页表项的实现方法。  
这个快速的方法就是get_pte_array(&self)-> &'static mut [PageTableEntry]，它可以根据一个物理页号PhysPageNum找到对应的物理地址，找到物理地址之后，它返回的是一个把pte形式的一个数组，我们的每一个页表都是放在一个整页里面，我们把这个整页看成一个数组，返回了一个pte的数组，这样的话我们想要访问里面的任何一个页表项pte，我们只要给索引就可以了。  
有时候可能我们访问物理页的时候不希望用pte的形式，他有可能是数据，这样的话我们一字节数组的形式get_bytes_array(&self) -> &'static mut [u8]去获取它可能会更有帮助。有时候有一些其他的情况的话，我们提供一个泛型T，get_mut<T>(&self) -> &'static mut T，然后我们可以任意选择把这一页解析成什么结构，然后我们也更方便的用这种结构去访问页里面的内容。  
*/  

### 分配空闲物理页帧与释放已用页帧

/*  
在此之前，都是一些虚拟页号，物理页号之间的一些转化和解析，实际上并没有真正的去探测我们的物理内存，现在我们要实现分配空闲物理页帧与释放已用页帧这样一个结构。
*/  

我们声明一个FrameAllocator Trait来描述一个物理页帧管理器需要提供哪些功能：
    //src/mm/frame_allocator.rs
    trait FrameAllocator {
        fn new() -> Self;
        fn alloc(&mut self) -> Option<PhysPageNum>;
        fn dealloc(&mut self, ppn: PhysPageNum);
    }

真实的分配器使用一个简单的栈式分配器
    //src/mm/frame_allocator.rs
    pub struct StackFrameAllocator {
        current: usize,
        end: usize,
        recycled: Vec<usize>,
    }

/*
如果一开始，系统里面所有的内存页都没有被用过的话，[current,end]就是从0到最大的边界。  
*/  
[current,end]表示从未使用过的且可用的物理页帧号，recycled用来存放被回收的物理页，当需要分配新的页帧的时候，优先从recycled中获取页帧，如果recycled为空，再从[current,end]区域分配可用帧。当页帧被回收的时候，将该栈帧推入recycled中，用代码描述其方式如下：  

    impl FrameAllocator for StackFrameAllocator {
        //之后会有初始化，现在设置成[0,0]没有事
        fn new() -> Self {
            Self {
                current: 0,
                end: 0,
                recycled: Vec::new(),
            }
        }
        fn alloc(&mut self) -> Option<PhysPageNum> {
            //如果recycled里面有的话，从里面拿
            if let Some(ppn) = self.recycled.pop() {
                Some(ppn.into())
            } else if self.current == self.end {
                //如果recycled没有，并且current到end了，就没有多的页可以用了，返回None
                None
            } else {
                //否则，从[current,end]里面拿current的
                self.current += 1;
                Some((self.current - 1).into())
            }
        }
        fn dealloc(&mut self, ppn: PhysPageNum) {
            let ppn = ppn.0;
            // validity check
            if ppn >= self.current || self.recycled.iter().any(|v| *v == ppn) {
                panic!("Frame ppn={:#x} has not been allocated!", ppn);
            }
            // 把要回收的页放到recycle里面去
            self.recycled.push(ppn);
        }
    }

    impl StackFrameAllocator {
        //new()出来之后会进行初始化
        pub fn init(&mut self, l: PhysPageNum, r: PhysPageNum) {
            self.current = l.0;
            self.end = r.0;
        }
    }

跟之前的方法类似，创建一个StackFrameAllocator的全局实例，然后给出各种操作的外部接口。注意，在全局实例中需要将其可用帧初始化为真实可用的页帧范围：
    type FrameAllocatorImpl = StackFrameAllocator;

    lazy_static! {
        /// frame allocator instance through lazy_static!
        //创建一个全局实例，并且用智能指针给包起来，才能更好的全局使用
        pub static ref FRAME_ALLOCATOR: UPSafeCell<FrameAllocatorImpl> =
            unsafe { UPSafeCell::new(FrameAllocatorImpl::new()) };
    }

    /// initiate the frame allocator using `ekernel` and `MEMORY_END`
    //对实例进行初始化的实现
    pub fn init_frame_allocator() {
        extern "C" {
            fn ekernel();
        }
        //我们把ekernel作为一个起始地址，MEMORY_END作为一个结束地址，这一段作为页帧分配器的使用范围
        //ekernel就是end kernel的意思，即是内核区的结束的位置，也就是说从start kernel到end kernel是内核占据的，我们不能用这些。  
        //MEMORY_END就是整个内存可以使用的最大的上界
        FRAME_ALLOCATOR.exclusive_access().init(
            PhysAddr::from(ekernel as usize).ceil(),
            PhysAddr::from(MEMORY_END).floor(),
        );
    }

    /// allocate a frame
    pub fn frame_alloc() -> Option<FrameTracker> {
        FRAME_ALLOCATOR
            .exclusive_access()
            .alloc()
            .map(FrameTracker::new)
    }

    /// deallocate a frame
    fn frame_dealloc(ppn: PhysPageNum) {
        FRAME_ALLOCATOR.exclusive_access().dealloc(ppn);
    }

注意到frame_alloc()的返回值为Option，其定义为：
    /// manage a frame which has the same lifecycle as the tracker
    //FrameTracker实际上就是ppn：PhysPageNum做了一层封装，我们做这层封装的原因是我们要Drop特性
    pub struct FrameTracker {
        pub ppn: PhysPageNum,
    }

    impl FrameTracker {
        pub fn new(ppn: PhysPageNum) -> Self {
            // page cleaning
            let bytes_array = ppn.get_bytes_array();
            for i in bytes_array {
                *i = 0;
            }
            Self { ppn }
        }
    }

    impl Debug for FrameTracker {
        fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
            f.write_fmt(format_args!("FrameTracker:PPN={:#x}", self.ppn.0))
        }
    }

    //我们的数据结构如果有了Drop，任何一个结构，它在超出了自己的生命周期之后，释放的时候就会调用drop函数，
    //而这里的drop会将刚才的物理页释放掉
    impl Drop for FrameTracker {
        fn drop(&mut self) {
            frame_dealloc(self.ppn);
        }
    }

这样做的好处是，在new()时，可以将对应的页帧清空，更重要的是，当Frame Tracker超出生命周期时，将自动释放掉对应的页帧，这个过程的实现是非常巧妙的，我们让一个PageTable的FrameTracker都属于该PageTable，而每个PageTable又最终属于一个进程，只需要让进程结束时释放页表，而页表释放时释放自身的FrameTracker，则就可以实现进程释放的时候其所占用的所有物理页帧都被释放掉。

### 在三级页表结构中访问和创建虚拟页表的页表项
第一个相关函数为：  
    fn find_pte_create(&mut self, vpn: VirtPageNum) -> Option<&mut PageTableEntry>

它能够返回vpn所对应的PTE，若在查询过程中发现其一级页号或二级页号在对应的页表中的页表项并不valid（该上级索引还没有被分配下级页表），则为其分配下级页表以便得到对应的PTE，其基本实现如下：  
    //src/mm/pagetable.rs
    impl PageTable {
        //功能：在页表当中找一个虚拟页对应的页表项，如果没有我们就创建页表项
        fn find_pte_create(&mut self, vpn: VirtPageNum) -> Option<&mut PageTableEntry> {
            //返回一个数组，分别代表一级索引、二级索引、三级索引
            let mut idxs = vpn.indexes();
            //根页表的起始地址
            let mut ppn = self.root_ppn;
            let mut result: Option<&mut PageTableEntry> = None;
            for (i, idx) in idxs.iter_mut().enumerate() {
                //先去根页表里面获取第idx项，找到了根页表里面对应的项pte
                let pte = &mut ppn.get_pte_array()[*idx];
                //i==2时break是因为我们的目标不是去找到对应的物理页，我们的目标只是找到或者创建它的表项
                //i==2的时候上一步pte做完就已经是那个表项了，我们只需要把它赋值给result返回就可以了
                if i == 2 {
                    result = Some(pte);
                    break;
                }
                //如果是前面的一级索引无效的话，说明还没有创建对应的二级页表
                if !pte.is_valid() {
                    //需要给它分配新的页帧
                    let frame = frame_alloc().unwrap();
                    //然后让他能够指向对应的页，并且修改有效位
                    *pte = PageTableEntry::new(frame.ppn, PTEFlags::V);
                    //然后要把新建的页帧放进去
                    self.frames.push(frame);
                }
                ppn = pte.ppn();
            }
            result
        }
    }

另一个类似的函数是：
    fn find_pte(&self, vpn: VirtPageNum) -> Option<&PageTableEntry>
其实现为：
    impl PageTable {
        fn find_pte(&self, vpn: VirtPageNum) -> Option<&PageTableEntry> {
            let idxs = vpn.indexes();
            let mut ppn = self.root_ppn;
            let mut result: Option<&PageTableEntry> = None;
            for (i, idx) in idxs.iter().enumerate() {
                let pte = &ppn.get_pte_array()[*idx];
                if i == 2 {
                    result = Some(pte);
                    break;
                }
                if !pte.is_valid() {
                    return None;
                }
                ppn = pte.ppn();
            }
            result
        }
    }
实际上它的含义同样为查找vpn对应的PTE，但是如果该PTE还没有在页表中被创建，则返回None。如果返回None则说明它的一级索引或二级索引出了问题；如果没有对应的物理页帧，还是会返回PTE，只是返回的PTE的有效位是无效的。  

配合一下两个函数我们可以不通过MMU而手动查询页表内容。
/*
我们的操作系统实际上可以去查每一个进程的页表，这个时候我们希望知道它对应的物理页是哪一个，所以我们需要提供一种手动查询的方式。
*/
    impl PageTable {
        //通过页表的基址satp（根目录的地址）获取一个新的页表结构
        //这个页表与原页表是一样的，我们在这个页表里面做查询工作
        pub fn from_token(satp: usize) -> Self {
            Self {
                root_ppn: PhysPageNum::from(satp & ((1usize << 44) - 1)),
                frames: Vec::new(),
            }
        }

        //获取页表项，然后复制，copy出来之后我们随意修改，他不会改变原来的页表项
        pub fn translate(&self, vpn: VirtPageNum) -> Option<PageTableEntry> {
            self.find_pte(vpn).copied()
        }
    }

实现这些功能后，我们提供了两个很重要的函数：
    #[allow(unused)]
    //为页表的虚拟页vpn增加一个对应的页表项,物理页号为ppn，标记位为flags。
    pub fn map(&mut self, vpn: VirtPageNum, ppn: PhysPageNum, flags: PTEFlags) {
        let pte = self.find_pte_create(vpn).unwrap();
        assert!(!pte.is_valid(), "vpn {:?} is mapped before mapping", vpn);
        *pte = PageTableEntry::new(ppn, flags | PTEFlags::V);
    }
    #[allow(unused)]
    //取消一个虚拟页vpn的映射
    pub fn unmap(&mut self, vpn: VirtPageNum) {
        let pte = self.find_pte_create(vpn).unwrap();
        assert!(pte.is_valid(), "vpn {:?} is invalid before unmapping", vpn);
        *pte = PageTableEntry::empty();
    }

最后，提供了一个辅助函数用来获取页表的根页表地址：
    pub fn token(&self) -> usize {
        8usize << 60 | self.root_ppn.0
    }

至此，页表相关的结构和方法均已经实现，我们对能被外层调用的接口做一个总结：
    impl PageTable{
        //创建空页表
        pub fn new() -> Self;
        //根据根目录地址satp创建根目录地址为satp的空页表，其实是对原来页表的拷贝
        pub fn from_token(satp:usize) -> Self;
        fn find_pte(&self, vpn: VirtPageNum) -> Option<&PageTableEntry>;
        find_pte_create(&mut self, vpn: VirtPageNum) -> Option<&mut PageTableEntry>;
        pub fn map(&mut self, vpn: VirtPageNum, ppn: PhysPageNum, flags: PTEFlags);
        pub fn unmap(&mut self, vpn: VirtPageNum);
        pub fn translate(&self, vpn: VirtPageNum) -> Option<PageTableEntry>;
        pub fn token(&self) -> usize;
    }

## 2、逻辑段与地址空间的构建
### 逻辑段的结构与方法
/*
连续的虚拟页，比如说虚拟页20到虚拟页200，合在一起的小区间，我们把它们叫vpn range，我们如果对它进行统一的管理，我们需要一个结构，就叫MapArea。
*/
    /// map area structure, controls a contiguous piece of virtual memory
    pub struct MapArea {
        //迭代器，元素为所有的虚拟页
        vpn_range: VPNRange,
        //记录映射关系
        data_frames: BTreeMap<VirtPageNum, FrameTracker>,
        //表示映射方式
        map_type: MapType,
        //逻辑段的访问权限
        map_perm: MapPermission,
    }

逻辑段代表了一段连续的虚拟页所构成的虚拟地址空间，其中：
    /// map type for memory set: identical or framed
    pub enum MapType {
        //表示对等映射
        Identical,
        表示对于每个虚拟页面都需要映射到一个新分配的物理页帧
        Framed,
    }

    bitflags! {
        /// map permission corresponding to that in pte: `R W X U`
        pub struct MapPermission: u8 {
            const R = 1 << 1;
            const W = 1 << 2;
            const X = 1 << 3;//执行
            const U = 1 << 4;//表示是不是在用户态下可以访问
        }
    }

MapArea与页表映射有关的方法为：
    impl MapArea{
        //创建一个新的MapArea
        pub fn new(
            start_va: VirtAddr,
            end_va: VirtAddr,
            map_type: MapType,
            map_perm: MapPermission,
        ) -> Self {
            let start_vpn: VirtPageNum = start_va.floor();
            let end_vpn: VirtPageNum = end_va.ceil();
            Self {
                vpn_range: VPNRange::new(start_vpn, end_vpn),
                data_frames: BTreeMap::new(),
                map_type,
                map_perm,
            }
        }
        //为MapArea中的一个虚拟页映射物理页，若为恒等映射，则直接映射到与vpn同号的ppn；否则重新分配一个物理页帧使之映射。映射完成后需要在页表中也完成映射操作。
        pub fn map_one(&mut self, page_table: &mut PageTable, vpn: VirtPageNum) {
            let ppn: PhysPageNum;
            match self.map_type {
                MapType::Identical => {
                    ppn = PhysPageNum(vpn.0);
                }
                MapType::Framed => {
                    let frame = frame_alloc().unwrap();
                    ppn = frame.ppn;
                    self.data_frames.insert(vpn, frame);
                }
            }
            let pte_flags = PTEFlags::from_bits(self.map_perm.bits).unwrap();
            page_table.map(vpn, ppn, pte_flags);
        }
        //解除映射
        #[allow(unused)]
        pub fn unmap_one(&mut self, page_table: &mut PageTable, vpn: VirtPageNum) {
            #[allow(clippy::single_match)]
            match self.map_type {
                MapType::Framed => {
                    self.data_frames.remove(&vpn);
                }
                _ => {}
            }
            page_table.unmap(vpn);
        }
        //完成对MapArea中所有虚拟页的映射
        pub fn map(&mut self, page_table: &mut PageTable) {
            for vpn in self.vpn_range {
                self.map_one(page_table, vpn);
            }
        }
        //解除对MapArea中所有虚拟页的映射，并且在某一个页表上把这个映射也消除掉
        #[allow(unused)]
        pub fn unmap(&mut self, page_table: &mut PageTable) {
            for vpn in self.vpn_range {
                self.unmap_one(page_table, vpn);
            }
        }
    }

在此基础上，可以实现在MapArea中存入数组数据的方法：
    impl MapArea{
        /// data: start-aligned but maybe with shorter length
        /// assume that all frames were cleared before
        pub fn copy_data(&mut self, page_table: &mut PageTable, data: &[u8]) {
            assert_eq!(self.map_type, MapType::Framed);
            let mut start: usize = 0;
            //虚拟页区间中的第一个虚拟页
            let mut current_vpn = self.vpn_range.get_start();
            let len = data.len();
            loop {
                //得到数组里面的数据
                let src = &data[start..len.min(start + PAGE_SIZE)];
                //找到物理页地址拷贝过去
                let dst = &mut page_table
                    .translate(current_vpn)//从页表里面找到对应的页表项
                    .unwrap()
                    .ppn()//对应的物理页
                    .get_bytes_array()[..src.len()];//把它变成了一个字节数组
                dst.copy_from_slice(src);
                //最多拷贝一个页
                start += PAGE_SIZE;
                if start >= len {
                    break;
                }
                current_vpn.step();
            }
        }
    }

该函数将数据放入范围内的每个页中，一个页存满了则进入下一个页存放，直到全部数据存放完毕或者MapArea中的虚拟页全部被用完。至此，我们可以以此为基础完成对进程地址空间的定义。

### 地址空间的结构与方法
地址空间是与进程一一对应的，其结构为：
    /// memory set structure, controls virtual-memory space
    pub struct MemorySet {
        page_table: PageTable,//一个地址空间对应一个页表
        areas: Vec<MapArea>,//一系列逻辑段
    }

它包含一个页表和一系列逻辑段，其基本方法包括：
    impl MemorySet {
        //创建一个空的地址空间
        pub fn new_bare() -> Self {
            Self {
                page_table: PageTable::new(),
                areas: Vec::new(),
            }
        }
        //获取内置页表的根目录地址satp
        pub fn token(&self) -> usize {
            self.page_table.token()
        }
        //获取内置页表对应虚拟页号vpn的页表项PTE
        pub fn translate(&self, vpn: VirtPageNum) -> Option<PageTableEntry> {
            self.page_table.translate(vpn)
        }
        //push方法将data写入逻辑段map_area中，再将map_area插入地址空间MemorySet
        //并且要将页表的映射也给完成了
        fn push(&mut self, mut map_area: MapArea, data: Option<&[u8]>) {
            //将map_area于pagetable绑定并分配物理页帧，完成了页表的映射
            map_area.map(&mut self.page_table);
            if let Some(data) = data {
                map_area.copy_data(&mut self.page_table, data);
            }
            self.areas.push(map_area);
        }
        /// Assume that no conflicts.
        //地址空间MemorySet中根据传入的参数插入一个没有存放数据的Frame类型的MapArea
        //在地址空间里面加入一串连续的地址映射
        pub fn insert_framed_area(
            &mut self,
            start_va: VirtAddr,
            end_va: VirtAddr,
            permission: MapPermission,
        ) {
            self.push(
                MapArea::new(start_va, end_va, MapType::Framed, permission),
                None,
            );
        }

    }

下面介绍一个特殊的函数：
    /// Mention that trampoline is not collected by areas.
    //功能：在自己的页表里面加入一个映射
    //会把虚拟页TRAMPOLINE和中断入口strampoline关联起来，也就是说，一个地址空间，如果我们给它指向了这个函数，我们可以利用TRAMPOLINE去访问中断strampoline。
    //在内置页表中，将虚拟地址TRAMPOLINE所对应的虚拟页映射到__alltraps所对应的页。
    fn map_trampoline(&mut self) {
        self.page_table.map(
            VirtAddr::from(TRAMPOLINE).into(),
            PhysAddr::from(strampoline as usize).into(),
            PTEFlags::R | PTEFlags::X,
        );
    }

我们注意到，这里出现了两个参数TRAMPOLINE和strampoline，我们分别看一下他们的定义：
    //src/config.rs
    //虚拟页里面地址最高就是usize::MAX，TRAMPOLINE因为减去一个PAGE_SIZE在次高页的位置上
    pub const TRAMPOLINE: usize = usize::MAX - PAGE_SIZE + 1;

    # src/linker.ld
    .text : {
        *(.text.entry)
        . = ALIGN(4K);//按页对齐
        strampoline = .;//strampoline一定是一个页的起点
        *(.text.trampoline);
        . = ALIGN(4K);
        *(.text .text.*)
    }

    #src/trap/trap.S
        .section .text.trampoline
        .globl __alltraps
        .globl __restore
        .align 2
    __alltraps://每次进入中断的时候

可以看到strampoline对应了__alltraps所对应的物理页的起始地址。

接下来，我们来实现内核载入时映射地址空间的函数：
    pub fn new_kernel() -> Self {
        let mut memory_set = Self::new_bare();
        // map trampoline
        //内核地址空间与中断的位置对应起来
        memory_set.map_trampoline();
        // map kernel sections
        info!(".text [{:#x}, {:#x})", stext as usize, etext as usize);
        info!(".rodata [{:#x}, {:#x})", srodata as usize, erodata as usize);
        info!(".data [{:#x}, {:#x})", sdata as usize, edata as usize);
        info!(
            ".bss [{:#x}, {:#x})",
            sbss_with_stack as usize, ebss as usize
        );
        //内核代码段对等映射
        info!("mapping .text section");
        memory_set.push(
            MapArea::new(
                (stext as usize).into(),
                (etext as usize).into(),
                MapType::Identical,
                MapPermission::R | MapPermission::X,
            ),
            None,
        );
        //内核rodata段对等映射
        info!("mapping .rodata section");
        memory_set.push(
            MapArea::new(
                (srodata as usize).into(),
                (erodata as usize).into(),
                MapType::Identical,
                MapPermission::R,
            ),
            None,
        );
        //数据段对等映射
        info!("mapping .data section");
        memory_set.push(
            MapArea::new(
                (sdata as usize).into(),
                (edata as usize).into(),
                MapType::Identical,
                MapPermission::R | MapPermission::W,
            ),
            None,
        );
        //bss段对等映射
        info!("mapping .bss section");
        memory_set.push(
            MapArea::new(
                (sbss_with_stack as usize).into(),
                (ebss as usize).into(),
                MapType::Identical,
                MapPermission::R | MapPermission::W,
            ),
            None,
        );
        //对整个内核到内存尽头这一段做了对等映射
        info!("mapping physical memory");
        memory_set.push(
            MapArea::new(
                (ekernel as usize).into(),
                MEMORY_END.into(),
                MapType::Identical,
                MapPermission::R | MapPermission::W,
            ),
            None,
        );
        memory_set
    }

可以看到，内核的地址空间对应了多个恒等映射的逻辑段。
/*
也就是说，对我们的操作系统来说，通过new_kernel出来的时候，整个地址空间都是一个对等映射，我们想访问哪个物理页我们直接写就行了，虚拟页就是物理页，虚拟地址就是物理地址。
*/

下面是载入elf文件的用户态程序的地址空间建立过程：
/*
实际上，虚拟地址在我们做内部链接的时候就已经形成了。  
*/
    pub fn from_elf(elf_data: &[u8]) -> (Self, usize, usize) {
        let mut memory_set = Self::new_bare();
        // map trampoline
        //不管是对内核还是用户来说，都是通过同样的一个物理地址来跳到中断这个位置
        memory_set.map_trampoline();
        // map program headers of elf, with U flag
        let elf = xmas_elf::ElfFile::new(elf_data).unwrap();
        let elf_header = elf.header;
        let magic = elf_header.pt1.magic;
        assert_eq!(magic, [0x7f, 0x45, 0x4c, 0x46], "invalid elf!");
        //应用程序在链接的时候就已经确定了每个数据的虚拟地址，在载入系统的时候，数据在程序中的虚拟地址和在虚拟内存中的虚拟地址是一致的，这样才能保证程序在进入虚拟内存系统之后，依然可以正常运行。
        let ph_count = elf_header.pt2.ph_count();
        let mut max_end_vpn = VirtPageNum(0);
        for i in 0..ph_count {
            let ph = elf.program_header(i).unwrap();
            if ph.get_type().unwrap() == xmas_elf::program::Type::Load {
                let start_va: VirtAddr = (ph.virtual_addr() as usize).into();
                let end_va: VirtAddr = ((ph.virtual_addr() + ph.mem_size()) as usize).into();
                let mut map_perm = MapPermission::U;//表示这个程序能够在用户态下访问
                let ph_flags = ph.flags();
                if ph_flags.is_read() {
                    map_perm |= MapPermission::R;
                }
                if ph_flags.is_write() {
                    map_perm |= MapPermission::W;
                }
                if ph_flags.is_execute() {
                    map_perm |= MapPermission::X;
                }
                let map_area = MapArea::new(start_va, end_va, MapType::Framed, map_perm);
                max_end_vpn = map_area.vpn_range.get_end();
                memory_set.push(
                    map_area,
                    Some(&elf.input[ph.offset() as usize..(ph.offset() + ph.file_size()) as usize]),
                );
            }
        }
        // map user stack with U flags
        //max_end_vpn是用户程序部分所占用的最后一个虚拟页，程序载入到了内存里面之后，除了程序的原本内容之外，我们还需要给它一些空间做用户栈。需要设置可读、可写、用户态下可用：MapPermission::R | MapPermission::W | MapPermission::U,  
        let max_end_va: VirtAddr = max_end_vpn.into();
        let mut user_stack_bottom: usize = max_end_va.into();
        // guard page
        //保护页；我们从它占了一个页的后一个页开始给它做用户栈
        user_stack_bottom += PAGE_SIZE;
        let user_stack_top = user_stack_bottom + USER_STACK_SIZE;
        memory_set.push(
            MapArea::new(
                user_stack_bottom.into(),
                user_stack_top.into(),
                MapType::Framed,
                MapPermission::R | MapPermission::W | MapPermission::U,
            ),
            None,
        );
        // map TrapContext
        //这一部分是预留了TRAMPOLINE前面的一个虚拟页来放置TRAP_CONTEXT
        memory_set.push(
            MapArea::new(
                TRAP_CONTEXT.into(),
                TRAMPOLINE.into(),
                MapType::Framed,
                MapPermission::R | MapPermission::W,
            ),
            None,
        );
        (
            memory_set,
            user_stack_top,
            elf.header.pt2.entry_point() as usize,
        )
    }

返回了一个三元组（进程的MemorySet，用户栈地址，入口地址）

至此，与MemorySet相关的方法均已实现了，我们用activate()函数来使其生效：
    //把寄存器给修改了
    pub fn activate(&self) {
        let satp = self.page_table.token();
        unsafe {
            satp::write(satp);//切换页表
            core::arch::asm!("sfence.vma");//返回TLB
        }
    }

实例化一个内核的地址空间：
    lazy_static! {
        /// a memory set instance through lazy_static! managing kernel space
        pub static ref KERNEL_SPACE: Arc<Mutex<MemorySet>> =
            Arc::new(Mutex::new(MemorySet::new_kernel()));
    }

    pub fn init() {
        heap_allocator::init_heap();
        frame_allocator::init_frame_allocator();
        KERNEL_SPACE.lock().activate();//处于内核的地址空间
    }

## 3、引入地址空间后的task设计
在chapter2和chapter3中任务想要运行必须将其手动载入特定的内存位置当中，并且通过设计task的TrapContext来跳转到对应的内存位置去执行。在本章节中，task可以通过虚拟页来访问对应的内容。因此只需要在存放的时候将存放程序的位置作为参数建立task的地址空间就可以了：
    //src/loader.rs
    /// get applications data
    pub fn get_app_data(app_id: usize) -> &'static [u8] {
        extern "C" {
            fn _num_app();
        }
        let num_app_ptr = _num_app as usize as *const usize;
        let num_app = get_num_app();
        let app_start = unsafe { core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1) };
        assert!(app_id < num_app);
        unsafe {
            core::slice::from_raw_parts(
                app_start[app_id] as *const u8,
                app_start[app_id + 1] - app_start[app_id],
            )
        }
    }

    //src/task/task.rs
    /// task control block structure
    pub struct TaskControlBlock {
        pub task_status: TaskStatus,
        pub task_cx: TaskContext,
        pub memory_set: MemorySet,//增加地址空间的设置
        //TrapContext在自身地址空间的物理页帧，根据恒等映射原则，内核可以用该地址访问TrapContext
        pub trap_cx_ppn: PhysPageNum,
        pub base_size: usize,//用户栈user_sp的位置
    }

    impl TaskControlBlock {
        //获取TrapContext
        pub fn get_trap_cx(&self) -> &'static mut TrapContext {
            self.trap_cx_ppn.get_mut()
        }
        //获取页表起始位置
        pub fn get_user_token(&self) -> usize {
            self.memory_set.token()
        }
    }

    impl TaskControlBlock {
        pub fn get_trap_cx(&self) -> &'static mut TrapContext {
            self.trap_cx_ppn.get_mut()
        }
        pub fn get_user_token(&self) -> usize {
            self.memory_set.token()
        }
        //创建一个新的进程
        pub fn new(elf_data: &[u8], app_id: usize) -> Self {
            // memory_set with elf program headers/trampoline/trap context/user stack
            let (memory_set, user_sp, entry_point) = MemorySet::from_elf(elf_data);
            let trap_cx_ppn = memory_set
                .translate(VirtAddr::from(TRAP_CONTEXT).into())
                .unwrap()
                .ppn();
            let task_status = TaskStatus::Ready;
            // map a kernel-stack in kernel space
            //获取内核堆栈地址，相关实现在src/config.rs
            let (kernel_stack_bottom, kernel_stack_top) = kernel_stack_position(app_id);
            //为用户程序的内核堆栈分配页帧，采用Frame模式
            KERNEL_SPACE.lock().insert_framed_area(
                kernel_stack_bottom.into(),
                kernel_stack_top.into(),
                MapPermission::R | MapPermission::W,
            );
            let task_control_block = Self {
                task_status,
                task_cx: TaskContext::goto_trap_return(kernel_stack_top),
                memory_set,
                trap_cx_ppn,
                base_size: user_sp,
            };
            // prepare TrapContext in user space
            let trap_cx = task_control_block.get_trap_cx();
            *trap_cx = TrapContext::app_init_context(
                entry_point,
                user_sp,
                KERNEL_SPACE.lock().token(),
                kernel_stack_top,
                trap_handler as usize,
            );
            task_control_block
        }
    }

值得注意的是，Task_Manager结构中存放TaskControlBlock的结构从本来的全局数组变更为了Vec;
    struct TaskManagerInner {
        /// task list
        tasks: Vec<TaskControlBlock>,
        /// id of current `Running` task
        current_task: usize,
    }

这是因为我们给出了动态分配的策略：
    #[global_allocator]
    /// heap allocator instance
    static HEAP_ALLOCATOR: LockedHeap = LockedHeap::empty();

    #[alloc_error_handler]
    /// panic when heap allocation error occurs
    pub fn handle_alloc_error(layout: core::alloc::Layout) -> ! {
        panic!("Heap allocation error, layout = {:?}", layout);
    }

    /// heap space ([u8; KERNEL_HEAP_SIZE])
    static mut HEAP_SPACE: [u8; KERNEL_HEAP_SIZE] = [0; KERNEL_HEAP_SIZE];

    /// initiate heap allocator
    pub fn init_heap() {
        unsafe {
            HEAP_ALLOCATOR
                .lock()
                .init(HEAP_SPACE.as_ptr() as usize, KERNEL_HEAP_SIZE);
        }
    }

其他关于task切换的方法基本上大同小异。

### 中断进入与中断恢复的设计

本实验采用双页表设计，这也就意味着用户态和内核态使用不一样的地址空间。当中断发生时，计算机会做一下工作：

sstatus的SPP字段会被修改为CPU当前的特权级（U/S）  
sepc会被修改为Trap处理完成后默认会执行的下一条指令的地址。  
scause/stval分别会被修改为这次Trap的原因以及相关附加信息。  
CPU会跳转到stvec所设置的Trap处理入口地址，并将当前特权级设置为S，然后从Trap处理入口地址处开始执行。

注意到，此时我们虽然位于内核态，但是依然处于用户进程的地址空间中，因此，我们需要把Trap处理的入口地址放入到用户地址空间中，即前面提到的：
    pub fn from_elf(elf_data: &[u8]) -> (Self, usize, usize) {
        let mut memory_set = Self::new_bare();
        // map trampoline
        memory_set.map_trampoline();
    }

再设置用户程序中断的入口地址：在中断返回的时候，把TRAMPOLINE设置成中断服务程序的地址。

中断处理入口处的代码功能为：保存完现场后，TrapContext中存放了kernel_satp,trap_handler,kernel_sp，在切换到内核堆栈中后，直接做对应的切换即可。这里使用了jr指令。这是一个绝对寻址的指令，可以确保跳转到目标位置。

处理完中断后，会调用trap_return(),之后会跳转到__restore继续执行，由于内核态和用户态下TRAMPOLINE都是一样的，所以页表切换后，这个执行过程并不会产生任何差错。

__restore包含了地址空间恢复和现场恢复。

值得注意的是，switch并没有太大的变化，因为switch并不涉及地址空间的切换，当我们修改current_task的时候，__restore函数自然会把地址空间切换到next_task。至此，chapter4的框架部分解析完成。

# 编程练习
## 问题一：重写sys_get_time和sys_task_info
引入虚存机制后，原来内核的sys_get_time和sys_task_info的函数实现就无效了。请你重写这个函数，恢复正常功能。
### 需要重写的原因
以sys_get_time为例，它的实现为：
    pub fn sys_get_time(_ts: *mut TimeVal, _tz: usize) -> isize {
        let _us = get_time_us();
        // unsafe {
        //     *ts = TimeVal {
        //         sec: us / 1_000_000,
        //         usec: us % 1_000_000,
        //     };
        // }
        0
    }

它传入的是一个TimeVal类型的指针，发起系统调用时是在用户态下传入的地址，在内核态下，该地址显然不再是传入的数据的位置，因此，我们只需要找到真实的地址位置即可。因此，我们只需要实现一个当前进程虚拟地址到物理地址的转换方法即可，一种实现方式为：
    pub fn translated_physical_address(token: usize, ptr: *const u8) -> usize{
        let page_table = PageTable::from_token(token);
        let mut va = VirtAddr::from(ptr as usize);
        let ppn = page_table.find_pte(va.floor()).unwrap().pnn();
        super::PhysAddr::from(ppn).0 + va.page_offset()
    }
由此，可以得到答案方法：

    pub fn sys_get_time(_ts: *mut TimeVal, _tz: usize) -> isize {
        let _us = get_time_us();
        let ts = translated_physical_address(_ts as *const u8) as *mut TimeVal;
        unsafe {
            *ts = TimeVal {
                sec: us / 1_000_000,
                usec: us % 1_000_000,
            };
        }
        0
    }

    pub fn sys_task_info(ti: *mut TaskInfo) -> isize {
        let _ti = translated_physical_address(ti as *const u8) as *mut TaskInfo;
        unsafe{
            *_ti = TaskInfo{
                status:get_current_status(),
                syscall_times:get_syscall_times,
                time:(get_time_us()-get_current_start_time())/1000,
            };
        }
        0
    }


## 问题二：mmap和munmap匿名映射
mmap 在 Linux 中主要用于在内存中映射文件， 本次实验简化它的功能，仅用于申请内存。

请实现 mmap 和 munmap 系统调用，mmap 定义如下：

fn sys_mmap(start: usize, len: usize, port: usize) -> isize
    syscall ID：222
    申请长度为 len 字节的物理内存（不要求实际物理内存位置，可以随便找一块），将其映射到 start 开始的虚存，内存页属性为 port
    参数：
            start 需要映射的虚存起始地址，要求按页对齐
            len 映射字节长度，可以为 0
            port：第 0 位表示是否可读，第 1 位表示是否可写，第 2 位表示是否可执行。其他位无效且必须为 0
    返回值：执行成功则返回 0，错误返回 -1
    说明：
            为了简单，目标虚存区间要求按页对齐，len 可直接按页向上取整，不考虑分配失败时的页回收。
    可能的错误：
            start 没有按页大小对齐
            port & !0x7 != 0 (port 其余位必须为0)
            port & 0x7 = 0 (这样的内存无意义)
            [start, start + len) 中存在已经被映射的页
            物理内存不足

munmap 定义如下：

fn sys_munmap(start: usize, len: usize) -> isize
    syscall ID：215
    取消到 [start, start + len) 虚存的映射
    参数和返回值请参考 mmap
    说明：
            为了简单，参数错误时不考虑内存的恢复和回收。
    可能的错误：
            [start, start + len) 中存在未被映射的虚存。

tips:
    一定要注意 mmap 是的页表项，注意 riscv 页表项的格式与 port 的区别。
    你增加 PTE_U 了吗？

### 增加映射函数
映射函数定义为：
    pub fn sys_mmap(start: usize, len: usize, port: usize) -> isize
我们观察到MemorySet里面有如下方法：
    pub fn insert_framed_area(
        &mut self,
        start_va: VirtAddr,
        end_va: VirtAddr,
        permission: MapPermission,
    ) {
        self.push(
            MapArea::new(start_va, end_va, MapType::Framed, permission),
            None,
        );
    }
这个方法与我们要实现的函数的参数基本一致，因此我们把它作为主题函数，同时不要忘了对参数进行类型检查，最终我们的实现方法如下：
    impl MemorySet{
        pub fn mmap(&mut self,start: usize, len: usize, port: usize) -> isize{
            //把地址变成了一个范围
            let vpnrange = VPNRange::new(VirtAddr::from(start).floor(), VirtAddr::from(start+len).ceil());
            //检查每个页是否已经被映射过了
            for vpn in vpnrange{
                if let Some(pte) = self.page_table.find_pte(vpn){
                    //对应页表项是否有效
                    if pte.is_valid(){
                        return -1;
                    }
                }
            }
            let mut map_prem = MapPermission::U;
            if (port & 1)!=0{
                map_perm|=MapPermission::R;
            }
            if (port & 2)!=0{
                map_perm|=MapPermission::W;
            }
            if (port & 4)!=0{
                map_perm|=MapPermission::X;
            }
            //把这一部分插入到地址空间里面去
            self.insert_framed_area(VirtAddr::from(start), VirtAddr::from(start+len), map_perm);
            0
        }
    }



    fn mmap(&self, start: usize, len: usize, port: usize) -> isize {
        //边界检查
        if (start % config::PAGE_SIZE != 0) || (port & !0x7 != 0) || (port & 0x7 == 0) {
            return -1;
        }

        let start_address = mm::VirtAddr(start);
        let end_address = mm::VirtAddr(start + len);

        let map_permission =
            mm::MapPermission::from_bits((port as u8) << 1).unwrap() | mm::MapPermission::U;

        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;

        for vpn in mm::VPNRange::new(mm::VirtPageNum::from(start_address), end_address.ceil()) {
            if let Some(pte) = inner.tasks[current].memory_set.translate(vpn) {
                if pte.is_valid() {
                    println!("[debug] This area is used!");
                    return -1;
                }
            };

            println!("[debug] {}", usize::from(vpn));
        }

        inner.tasks[current].memory_set.insert_framed_area(
            start_address,
            end_address,
            map_permission,
        );

        for vpn in mm::VPNRange::new(mm::VirtPageNum::from(start_address), end_address.ceil()) {
            if let None = inner.tasks[current].memory_set.translate(vpn) {
                return -1;
            };
        }

        return 0;
    }
此后，由系统调用函数直接调用当前进程的MemorySet的该方法再加上一些必要的边界检查即可。

同样，我们也可以在MemorySet中加入munmap方法：

    impl MemorySet {
        pub fn munmap(&mut self, start: usize, len: usize) -> isize{
            let vpnrange = VPNRange::new(VirtAddr::from(start).floor(), VirtAddr::from(start+len).ceil());
            //检查已经没有被映射
            for vpn in vpnrange{
                let pte = self.page_table.find_pte(vpn);
                if pte.is_none() || !pte.unwrap().is_valid(){
                    return -1;
                }
            }
            for vpn in vpnrange{
                for area in &mut self.areas{
                    if vpn < area.vpn_range.get_end() && vpn >= area.vpn_range.get_start(){
                        area.unmap_one(&mut self.page_table, vpn);
                    }
                }
            }
            0
        }
}


    fn munmap(&self, start: usize, len: usize) -> isize {
        if start % config::PAGE_SIZE != 0 {
            return -1;
        }

        let start_address = mm::VirtAddr(start);
        let end_address = mm::VirtAddr(start + len);

        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;

        for vpn in mm::VPNRange::new(mm::VirtPageNum::from(start_address), end_address.ceil()) {
            if let None = inner.tasks[current].memory_set.translate(vpn) {
                return -1;
            };

            if let Some(pte) = inner.tasks[current].memory_set.translate(vpn) {
                if pte.is_valid() == false {
                    return -1;
                }
            };
        }

        for vpn in mm::VPNRange::new(mm::VirtPageNum::from(start_address), end_address.ceil()) {
            inner.tasks[current].memory_set.munmap(vpn);
        }

        for vpn in mm::VPNRange::new(mm::VirtPageNum::from(start_address), end_address.ceil()) {
            if let Some(pte) = inner.tasks[current].memory_set.translate(vpn) {
                if pte.is_valid() {
                    println!("[debug] This area is used!");
                    return -1;
                }
            };
        }

        return 0;
    }
至此，chapter4的编程实验已经完成了。