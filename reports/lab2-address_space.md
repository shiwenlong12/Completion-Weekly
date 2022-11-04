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

一个usize正好对应64位二进制，因此可以代表一个页表项，我们的重点是构建页表项的处理函数：new(ppn: PhysPageNum, flags: PTEFlags) -> Self、empty() -> Self、ppn(&self) -> PhysPageNum、flags(&self) -> PTEFlags、is_valid(&self) -> bool、  readable(&self) -> bool、writable(&self) -> bool、executable(&self) -> bool。  
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


