# 实验目的
在之前的实验里，我们所有的数据、程序的访问都是内存当中，包括我们的每一个应用，我们如果想要执行它，都是先把数据硬编码到我们的一个操作系统的数据段里面去，然后当我们需要的时候我们去数据段里将它们取出来放到代码段中程序执行的位置去执行。显然当我们的应用非常多、非常大的时候，这种做法会让我们的内核的负担增大，所以我们希望像真实的操作系统一样有一个文件系统，我们的这些设备存放在外存当中，在需要的时候将它从外存里面读到内存里面来执行。  
实现了一个简单文件系统'easy-fs'，能够对持久储存设备I/O资源进行管理，具体来说就是：  
一：以文件为单位来组织磁盘（虚拟磁盘，是指持久储存设备）中的一系列数据信息  
二：以文件名来访问文件，并且能够根据文件地址来访问对应的数据。  
具体来说就是根据文件的路径path找到对应的文件，假设我们这个文件非常大，在用户看来是连续的，但是我们的操作系统把它存放在磁盘上的时候，它不一定能够找到连续的那么大的空间来存放，这个时候操作系统需要在磁盘上进行离散式的存放，而我们需要一种机制能够帮助我们从这个文件地址找到磁盘上对应的数据。例如这个文件是一个非常长的视频，我们要找到第32秒的那帧画面，第32秒就是文件地址，通过第32秒找到磁盘上的位置，在找到那帧画面就是对应的数据。  

# 实验过程分析：
因为文件系统是面向对象的，所以本次实验采用了自上而下的分析方法，即先考虑分析需要哪些功能，在考虑为了支撑这些功能需要哪些底层的支持，直到完成整体设计。
## 一：需要提供的系统调用
文件调用系统我们需要对外提供的系统调用接口为以下3种：
sys-open接口的功能是：根据文件名找到对应的文件，并且将该文件的访问权限交给进程，并且用返回的整数文件描述符作为它的访问标志；参数是路径path和flags；返回值是该文件的文件描述符。  
在正常的操作系统中，会有目录套目录的形式出现，但是在这次实验中，只有一个根目录。  
sys_reade和sys_write接口的功能分别是：进程通过打开文件时得到的文件描述符fd将文件写或者读到用户的buf数组中（涉及到地址映射）；参数为fd,缓冲区buf数组,读写长度len，如果读写长度不够就尽可能的进行读写，即当文件的字节多于缓冲区的字节时，只会读取缓冲区长度的字节；返回值是实际读写长度。    
sys-close功能为关闭进程对文件描述符为fd的文件的访问权限。  

## 二：磁盘、操作系统和进程对文件的读写访问的结构和方法。  
### 主要设计思路
   我们假设在磁盘中存放着一种结构DiskInode,我们只需要找到这个结构并且把它读入内存来分析，就可以获取这个文件的全部信息，并且可以通过随机访问的方式去访问文件的任意一个位置的内容。DiskInode被组织在外存中的一块叫DiskInode表的外存区域中，通过编号就可以唯一访问到。  
    如果我们有了DiskInide,sys_read和sys_write就都可以实现了，因为我们假设找到DiskInide就可以随机访问，DiskInide一个一个的放在外存的DiskInide表当中，所以每一个DiskInide都会有一个索引，所以我们根据这个索引就可以找到对应的DiskInide，DiskInide表相当于一个小缓存池，当我们需要的时候从里面找一个没有被分配的位置把它分配给DiskInide来用，用完了之后我们就利用标志位把它给释放掉。  
   DiskInode被存放在磁盘上，我们访问它就需要先把它放入内存中，DiskInode在内存中的结构我们称为inode，通过inode我们可以访问到其对应的DiskInode。因此进程想要获得对文件的访问能力就是要找到文件对应的inode,如果文件的DiskInode还没有被放入内存中，我们需要先将其处理一下变成inode，然后把inode缓存到内存中；如果文件的DiskInode已经被放入内存中，我们只需要直接返回其缓存在内存中的inode即可。  
   外存的结构是文件的一个固有属性，但是当我们把它拿到内存当中之后，我们还需要给它添加一些固有属性之外的别的属性，因为拿到内存当中是给程序用的。  
   因为一个DiskInide代表一个文件，并且访问一个文件的进程可能有多个，所以可能有多个进程访问同一个DiskInode,而对应的Inode只有一个，所以在C语言实现中我们需要给inode添加引用计数，但是在rust中我们只需要让Inode使用rust的Arc封装就可以很好的利用这个智能指针的特性。rust中编译器会自动实现引用计数加一，引用计数减一，释放inode等操作。  
   而每一个进程对文件访问时，都会有一些区别于其他进程的具体特征（例如进程的访问权限，进程当前的读写位置等），所以具体到进程我们同样需要建立一个结构OSInode给每一个进程使用。记录每个进程使用文件的特性，例如当前进程读到哪里了，方便进程切换之后能够继续此进程。  
### 代码的实现：
   我们需要找到DiskInide就能够随机读写文件当中的数据。所以实现DiskInide结构需要read_at和write_at函数，这两个函数的功能分别是读写文件。  
   设计Inode结构的目的是进程通过Inode来读写外存，所以实现Inode结构需要read_disk_inode和modify_disk_inode函数，这两个函数可以直接通过Inode对其对应的DiskInide并获取其返回值，例如read_at()和write_at()，利用这个方法我们可以实现利用Inode读写函数。这两个函数的参数分别是&self和闭包f，闭包f可以理解为一个函数，闭包里面传入DiskInide的引用，返回一个V，f可以取具体的函数。modify的闭包的参数需要可以修改。DiskInide可以read_at和write_at，所以Inode必须继承DiskInide的方法，参数offset表示读写的时候起始点的文件地址，参数buf有一个自己的长度，在Inode结构中read_at函数中，闭包disk_inode.read_at的结果作为read_disk_inode的结果作为read_at的结果。  
   OSInode的结构非常显然，有readable,writeable,inner里面是offset,inode；包含读写权限，起始地址offset以及inode指针，在读写函数的时候我们并没有给出文件的起点，实际上这个起点由各进程对应的OSInode的offset给出。OSInode结构需要read()和write()，read()需要从Inode里面取read_at,读完之后offset还需要加上读的长度。
   显然，在这个过程中实现了访问其内部对应的Inode，至此，只要获取了进程文件所对应的OSInode，我们就可以对文件进行读写操作了。而在进程的syscall中，我们查找OSInode的依据是一个自然数fd,这是因为我们为每一个进程维护一个OSInode表，fd对应的是OSInode在这个表中的索引，这个表位于进程控制块TCB中。我们用户用的是底层封装好的东西，这样层层封装对我们的逻辑有很大的帮助。  
   至此，我们就可以实现sys_read和sys_write了。  
   
## 三：通过文件名获取对文件的访问能力  
   我们实现了只要获取了进程文件所对应的OSInode，我们就可以对文件进行读写操作了，但是文件系统需要的是给了一个文件名我们就需要能够访问到对应的数据，所以我们就需要根据一个文件名来造出来一个OSInode给进程来用。  
   根据上述描述，我们知道，只要我们把文件的DiskInide最终转化为OSInode并存放在进程的fd_table中，并获取其id，就可以实现对文件数据的随机访问了，而open()操作实际上就是实现这个过程。  
### 目录与目录项
   想要实现open（）的功能，我们就必须要获取文件对应的DiskInide编号。因为获取了DiskInide编号我们就能找到对应的DiskInide，找到了DiskInide我们就能够封装一个Inode结构出来，有了这个Inode我们就能封装一个OSInode结构，然后有了这个OSInode结构放到表里面去，任务就完成。
   这里我们设计了一个特殊的结构：目录项DirEntry，这个结构包含一个文件名name和一个对应的索引号inode_number，因此我们只需要找到name为给定名称的DerEntry就可以了。然而，文件系统中的DerEntry可能有很多，而且需要可持久化的存储，我们不可能把DerEntry放在内存中，因此，我们把这些DerEntry也放在了一个文件中，并且将这个文件的DiskInide放在DiskInide表的0号位置，这个用来放置DerEntry的文件叫做目录，在我们的实验中只有一个目录文件，即根目录文件，它的DiskInide编号为0号，我们随时都可以访问到它。而通过遍历这个文件，我们就可以找到name为我们需要的名称的对应的DerEntry了。  
   DerEntry结构包含的方法有：as_bytes（）和as_bytes_mut（）访问并修改它们；name(),inode_number()。
   至此，我们可以为Inode结构提供检索对应文件名的Inode编号的方法。在Inode结构里面进行这项操作的原因是根目录的DiskInode的编号是固定的0号，find_inode_id()功能是从根目录里面找到name对应的目录项，参数是&self,name,disk_inode;实现方法是遍历了根目录里面的所有目录项，如果目录项的名字等于传入的参数name，那么我们就返回这个目录项的编号。函数find（）功能：返回Inode;参数是&self,name,&str；要返回一个Inode形式，实现是将self对应的disk_inode放进去，然后通过find_inode_id(name,disk_inode)获取一个对应的编号。  
   通过在根目录下调用find()方法，我们就可以获取到对应文件的一个Inode了，而根目录的编号已知为0，因此可以很简单的访问到。
   
## 四：文件系统的磁盘访问接口
   在之前我们都是假设能够随机访问磁盘，在这一节我们就要解决如何利用DiskInode去访问文件信息的问题。
### 块设备接口层
   一个文件系统对应一个虚拟设备，而我们对磁盘的访问是以Block为基本单位的，我们通过块的编号去读写整个磁盘块，我们通过虚拟化来实现对磁盘的这种访问模式，其具体细节在easy_fs_fuse中，我们利用一个Linux文件来模拟磁盘区域，但是现在我们只需要关注虚拟化为我们提供的接口即可：在BlockDevice结构中函数read_block(&self,block_id:usize,buf:&mut[u8])功能是根据block_id把一个我们规定为4096bit大小的磁盘块数据读到buf数组里面去，writeck(&self,block_id:usize,buf:[u8])功能是把buf中的数据写入block_id号磁盘块中。因此，我们只要有block_id我们就可以直接找到对应的block，然后把它读写，与buf进行数据交换。
### 块缓存层
   访问磁盘块时，我们有必要把最近访问的磁盘块在内存中缓存起来来减少I/O操作，从而提高系统性能。
   pub struct BlockCache{cache:[u8;BLOCK_SZ],block_id:usize,block_device:Arc<dyn BlockDevice>,modified:bool,},cache是内存里面的一个数组，作为高速缓存区；block_id代表这个缓存结构存放的是第几个block；block_device表示它属于哪个虚拟设备，这个虚拟设备在我们这里只有一个磁盘0号设备；modified表示修改位，即有没有被写过，如果这个缓存被写过，那它被释放掉的时候也应该真的去写磁盘read_block，如果这个缓存没有被写过，那么就没有必要写回到磁盘里面去了，可以直接把这个缓存给释放掉。
   方法fn addr_of_offset(&self,offset:usize) -> usize{}缓存的偏移；get_ref()获取磁盘块某一个偏移位置的指针；get_mut()可以去修改偏移位置的指针；read(),modify()分别进行读和修改操作；sync()把缓冲块的内容写回到磁盘里面去。
   我们在new()的时候已经把磁盘块从磁盘里面读到读到内存里面来了，所以我们后面访问磁盘都不去真正的访问磁盘块了，我们都是去访问这个缓存区，我们看这个缓存区里面有没有对应的磁盘块，如果有那我们就直接读这个磁盘块就可以了，如果没有的话，那我们把这个磁盘块从外存当中读到缓存里面来，然后我们在访问这个磁盘块。所以，从后面开始我们的接口就需要变化了，read_block()与write_block()这两个函数我们就直接不需要使用了。至此，对磁盘块的访问完全转化为了对缓存块的访问，与之前类似，我们设计一个结构来组织它，并为其提供接口。
   pub struct BlockCacheManager{queue:VecDeque<(usize,Arc<Mutex<BlockCache>>)>,}实现方法get_block_cache()就是把这个磁盘块拿过来，然后我们从里面读从里面写，相当于对磁盘块的访问。写的时候可能有一个一致性问题，就是写一半的时候断电了，磁盘里面的数据就没了，但是这里我们暂时不考虑这么多。
   然后就是常规的实例化一个BlockCacheManager并且向外暴露两个接口：get_block_cache()获取对应的缓存块和block_cache_sync_all()功能是把整个缓存区全部写回磁盘块中。

## 五：文件系统外存布局
   在之前的内容中，我们自需要知道DiskInode的编号就能够知道DiskInode在哪里，这是因为查表了，那我们起码要知道这个表的起始位置在哪里，然后这个表要控制哪些磁盘块，它要知道每个数据在第几个磁盘块上面，所以我们起码要知道磁盘块区域在哪里，比方说是在第1024个磁盘块磁盘块开始的，那我说第三个数据块就是第1024+3=1027个数据块，这样我们才能去访问到它。所以我们需要知道磁盘的布局，easy-fs磁盘按照块编号从小到大的顺序依次分成了连续的5个区域分别是：  
   第一个区域只包括一个块，即超级块（Super Block），用于定位其他区域的连续位置，检查文件系统的合法性。  
   第二个区域是一个索引节点位图（inode bitmap），长度为若干个块，它记录了索引节点区域中有哪些索引节点已经被分配出去使用了。  
   第三个区域是索引节点区域（inode area)，长度位若干个块，其中每个块都存储了若干个索引节点。  
   第四个区域是一个数据块位图（data bitmap)，长度位若干个块，它记录了后面的数据块区域中有哪些已经被分配出去使用了。  
   最后的区域是数据块区域（data area），其中的每个被分配出去的块保存了文件或目录的具体内容。   
### 超级块
   超级块结构为pub struct SuperBlock{magic:u32,pub total_blocks:u32,pub inode_bitmap_blocks:u32,pub inode_area_blocks:u32,pub data_bitmap_blocks:u32,pub data_area_blocks:u32,}  
### 位图
   位图就是一个区域，比如说有1024个磁盘块，我们要知道它有没有被使用，如果是空闲的，那么我们就有1024个bit的位图去对应1024个磁盘块，然后我们查表，发现它的bit是0的时候就代表对应的磁盘块没有被使用，如果等于1，那就代表被使用了。这个位图的存在对我们磁盘分配很有用，我们释放磁盘块的时候，只需要把磁盘块对应的位图改为0就可以了。  
   位图结构是pub struct Bitmap{start_block_id:usize,blocks:usize,},位图也存放在磁盘中，如果我们知道了一个位图的起始位置，和这个位图有多大能控制多少块，那么我们在磁盘当中就能够找到这个位图，然后我们对位图定义提供分配和回收对应数据块的方法：new(),alloc():从这个位图里面去找第一个即对应bit位为0的哪个位置，找到对应的磁盘块,获取对应的磁盘块编号，dealloc()控制Block的回收。  
###  磁盘索引节点DiskInode  
   在这一部分我们就要构建DiskInode结构和实现方法，来实现如何通过DiskInode来访问文件这一个操作的。  
   DiskInode的结构是pub struct DiskInode{pub size:u32,pub direct:[u32;INODE_DIRECT_COUNT],pub indirect1:u32,pub indirect2:u32,type_:DiskInodeType,},size表示对应的文件的大小，direct直接索引，indirect1一级索引，indirect2二级索引，type_表示它是文件还是目录，在我们这次实验中，只有一个根目录其他的type_都是文件。  
   每一个文件/目录在磁盘上都是以一个DiskInode的形式存储，其中包含文件/目录的元数据：size表示文件目录的字节数;type_表示索引节点的类型DiskInodeType，目前仅支持文件File和目录Directory;其余的direct，indirect1，indirect2都是存储文件内容/目录内容的数据块的索引。  
   根据之前的假设，DiskInode应该具备随机访问文件随机地址的接口。  
   我们的磁盘数据都是以Block格式来进行存放的，那么我们逻辑上的文件也应该有逻辑上的块，如果我们能够完成逻辑块到物理块之间的一个转化，那么我们就能够完成地址的访问，就跟我们的虚拟页表是一样了，所以就有了接口get_block_id(&self,inner_id:u32,block_device:&Arc<dyn BlockDevice>),inner_id表示的是逻辑上的第多少块，我们文件的存放方式是先用直接索引direct，如果直接索引不够用那么就用一级索引indirect1，一级索引还不够用那么就用二级索引indirect2，所以如果inner_id小于直接索引的个数INODE_DIRECT_COUNT,那么我们直接返回inner_id对应的直接索引就可以了。如果inner_id大于直接索引的个数INODE_DIRECT_COUNT，小于一级索引的个数INDIRECT1_COUNT,那么我们就用一级索引，通过get_block_cache()先找到一级索引块，看看一级索引块里面存的是什么，然后再从里面找到对应的，假设之前的直接索引有1000个，那么第1006个就只能放在一级索引里面，就是一级索引里面的第6块，想要找到一级索引里面的第6块，就需要先看看一级索引指向的是哪个索引块，然后需要去这个索引块里面找到它存放的第6个索引，然后它就是那个对应的指针对应的数据块所在的索引的索引块的位置。  
   通过这个接口，可以获取文件逻辑块对应的物理块的地址，然后类似于虚拟页表中的转化方法就可以访问到对应的物理地址了，接下来解决了如何实现read_at(&self,offset:usize,buf:&mut[u8],block_device:&Arc<dyn BlockDevice>,),write_at的问题。read_at()函数先利用offset找到一个块，然后我们实现块对块的一个转化，文件的虚拟块到实际的物理块之间的转化，转化完了之后，找到磁盘当中的物理地址进行读写就可以了。write_at()和read_at()基本一样，不过在write_at()之前需要对比缓冲区和文件容量的大小，如果容量不够需要调用扩容函数increase_size(&mut self,new_size:u32,new_blocks:Vec<u32>,block_device:&Arc<dyn BlockDevice>,),在这里也涉及到如果直接索引够用怎么样，一级索引够用怎么样，二级索引够用怎么样，所以需要进行一系列的比较，然后决定从哪里去分配给它索引块，然后给它分配数据块，然后还要填入到对应的索引指针的位置和索引块里面去。与increase_size()对应的还有clear_size(&mut self,block_device:&Arc<dyn BlockDevice>,)函数，就是文件被清空的时候，它所有的磁盘块都要回收掉。  
   到这里，我们就实现了通过DiskInode对文件进行随机访问了。  
   
## 六：用户read()和write()整体逻辑整理
### 从OSInode到读取文件数据
   以read()为例，read()的接口为pub fn sys_read(fd:usize,buf:*const u8,len:usize) -> isize,它需要获取文件的OSInode，然后利用OSInode的接口read()来读取，而OSInode是借助其内部封装的Inode来read的，然后我们可以发现Inode的read_at是通过DiskInode的read来实现的，最后，我们在磁盘索引节点DiskInode中提到了具体利用DiskInode去读取文件的方法，至此，完成了闭环。  
### 获取OSInode的方法  
   另一条主线是把文件对应的OSInode放入进程的fd_table中，参数为文件的path，这里使用系统调用open();主体函数是open_file(name:&str,flags:OpenFlags) -> Option<Arc<OSInode>>，这个函数比较重要的点是:参数flags是一个标记，如果参数flags含有一个CREATE，那么如果找到了ROOT_INODE.find(name)返回的inode，那么就封装好返回OSInode,如果没有找到find(name)返回的inode，那么就调用ROOT_INODE.create(name)创建一个，然后同样封装好返回给open_file函数；如果参数flags没有一个CREATE，那么如果找到了find(name)返回的inode，那么就封装好返回OSInode给open_file函数，如果没找到就直接返回NONE。  
   主体函数是根节点下的find()和create()函数，这里先看find(&self,name:&str) -> Option<Arc<Inode>>函数这条路线，ifnd()函数是Inode结构的方法，这个方法的重点是通过自己的方法read_disk_inode获取到find_inode_id()，然后通过find_inode_id()返回值和get_disk_inode_pos(inode_id)获取到对应的位置。  
   函数find_inode_id(&self,name:&str,disk_inode:&DiskInode,) -> Option<u32>需要对目录项进行遍历，在遍历的时候,需要的时候就会去访问到DiskInode,然后用到disk_inode.read_at(DIRENT_SZ*i,dirent.as_bytes_mut(),&self.block_device,),read_at()根据直接索引，一级索引，二级索引来实现。  
### 创建文件的方法
   刚才提到了在inode代表的文件目录下面创建一个新文件的接口create()，创建一个文件，自然需要在DiskInode表中新分配一个DiskInode，获取其编号，然后创建它的目录项，使该目录项指向新分配的DiskInode，就是把刚才获得的编号写到这个新创建的目录项里面去，然后还要再目录文件下面插入该目录项。所以create()函数整体上就是按照这样来工作的。  
   函数pub fn create(&self,name:&str) -> Option<Arc<Inode>>首先需要先找到root_inode，然后从root_inode里面去找到对应的目录项，如果找到了，那么就返回NONE，因为root_inode里面已经有这个名字了，我们不能去创造同名的目录项；如果没有找到的话，那么就先使用new_inode_id = fs.alloc_inode()分配一个DiskInode,然后去到对应的磁盘的数据位置得到new_inode_block的编号id和偏移位置offset，然后我们想要访问磁盘块，就需要通过缓冲get_block_cache()，在里面进行一些必要的初始化，然后我们再把目录项给写如目录中，最后，我们需要对它进行同步，就是我们刚才写进去了之后我们需要把它写回到对应的DiskInode里面去，这样，我们就完成了create()。

## 七：对文件系统的最外层封装
   定义一个文件系统，实际上只需要给出一个虚拟设备名以及其基本磁盘布局就可以了。
   pub struct EasyFileSystem{pub block_device:Arc<dyn BlockDevice>,pub inode_bitmap:Bitmap,pub data_bitmap:Bitmap,inode_area_start_block:u32,data_area_start_block:u32,}描述了这个文件系统首先是哪个设备，然后它的索引和数据个位图，然后索引和数据的起始区域。假设有一个设备，然后这个设备里面该有的区域我们都已经知道了，那么实际上我们已经知道了这个文件系统，因为它的接口我们已经有了，通过EasyFileSystem结构我们知道哪一块是索引inode哪一块是数据data，哪一块是位图bitmap，然后根据接口，我们就能够访问里面的文件了。但是我们既然外层封装了一层，那么我们当然需要进行进一步的封装处理。  
   在EasyFileSystem结构中，pub fn create(block_device:Arc<dyn BlockDevice>,total_blocks:u32,inode_bitmap_blocks:u32) ->Arc<Mutex<self>>参数为设备，总块数量，索引位图的块个数，我们知道索引位图有多少个块我们就可以知道索引有多少个块，然后我们又知道一共有多少个块，两者相减，我们就能够知道数据有多少个块，这样的话整个区域我们就全都清楚了，就可以根据外存布局，完成create()函数。pub fn open(block_device:Arc<dyn BlockDevice>) -> Arc<Mutex<self>>函数的功能就是将文件系统载入我们的机器，即把文件系统的超级块SuoerBlock放入我们的内存里面去，这样的话我们就可以通过超级块做各种各样的访问。pun fn root_inode(efs:&Arc<Mutex<self>>) ->Inode功能是返回文件系统的根目录。pun fn get_disk_inode_pos(&self,inode_id:u32) ->(u32,usize)功能是前面封装好的，通过id获取inode的位置，返回值有两个，一个是它的磁盘号，一个是它的偏移量。   get_data_block_id(&self,data_block_id:u32) -> u32功能是通过数据块编号得到数据块位置，是很简单的加法。alloc_inode(&mut self) -> u32分配一个新的索引；alloc_data(&mut self) -> u32分配一个新的数据块；dealloc_data(&mut self,block_id:u32回收一个数据块。  
   
## 八：将应用打包到文件系统中
   这个过程实际上就是在宿主OS上创建一个文件，在这里就是创建了一个Linux文件，大小是4MB，将其封装为块设备，并将read_block()和write()的接口暴露给OS，我们以该f为虚拟设备构建一个EasyFileSystem结构，此后，我们遍历每一个应用的文件名，用它作为name来创建文件，再在该文件中写入对应的程序数据，也就是用文件名创建目录，然后我们用write把实际的数据写到这个文件里面去。这样，在系统视角下，我们就可以通过文件名来访问所有的应用，而不必像之前的实验那样把所有的数据都硬编码在内核的数据段中。  
   
# lab4编程作业
## 硬链接
   硬链接的含义就是我们有一个目录项指向一个文件，现在我们创建它的一个硬链接就是我们再创造一个目录项也指向这个文件，
   硬链接要求两个不同的目录项指向同一个文件，在我们的文件系统中也就是两个不同名称的目录项指向同一个磁盘块。本节要求实现三个系统调用：sys_linkat,sys_unlinkat,fstat。
   fn linkat(olddirfd:i32,oldpath:*const u8,newdirfd:i32,newpath:*const u8,flags:u32) -> i32功能是创建一个文件的硬链接。
   fn unlinkat(dirfd:i32,path:*const u8,flags:u32) ->i32功能是取消一个文件路径到文件的链接。
### sys_linkat
   在这次实验中，pub fn sys_linkat（oldpath:*const u8,newpath:*const u8） -> isize,语义就是创建一个新的目录项，name为newpath，将其链接到oldpath对应的DiskInode上。它的基本思路很简单，通过oldpath检索对应的inode编号n，再找到对应的DiskInode，将这个DiskInode的链接计数加1，然后创建一个新的目录项，新目录项的name是newpath，索引节点的编号为n，就可以完成任务，将新的目录项链接上了。主要通过link_file()函数来实现,link_file(oldname:&str,newname:&str) -> Option<()>在根目录下实现ROOT_INODE.link(oldname,newname)。
   重点就是Inode结构里的函数实现，link(&self,oldname:&str,newname:&ste) -> Option<()>实现方法是首先获取old_inode_id，如果没有旧的id，那么就是失败了直接返回NONE，然后获取旧的id对应的DiskInode的块编号block_id和块偏移量block_offset，然后就能够将DiskInode的链接计数加一n.nlink += 1,然后通过modify_disk_inode修改root_inode，向里面write_at，从原文件的末尾处开始写，写我们新创建的这个目录项。
### sys_unlinkat
   sys_unlinkat(path：*const u8) -> isize功能是将path对应的硬链接删除。
   unlinkat稍微复杂一点，因为检索到path对应的DiskInode，将其所对应的链接计数减一之后，还需要从根目录中把名为path的目录项移除,这个移除的实现比较复杂。还有就是如果链接计数为0了之后，我们需要去把这个DiskInode去释放掉，好在释放函数我们都已经写好了。
   具体实现与linkat一样，主要关注unlink(&self,name:&str) -> Option<()>，删除目录项的实现方法是新建一个可变向量V，我们需要找到文件名等于传入参数name的目录项，并且把它删掉，我们把其他的不用被删除的目录项全部用向量V保存起来，然后我们把V里面的内容拿给根目录重写一下，就实现了删除指定目录项的功能。实现就是先访问到根目录里面，然后同样是检索，如果检索的目录项的名字不是传入的名字，那么我们就把这个目录项push到V中，如果是传入的name，那么我们就可以获取到它里面的哪个inode也就是文件对应的inode，把inode写在inid里面。然后我们再去根目录里面，我们需要把根目录给dealloc掉，然后我们把V重新写进去，写完之后，我们需要把n.nlink -=1,如果n.nlink==0，也就是引用计数没了，那么我们需要把整个全部清空，做完了这些只是get_block_cache对缓冲区做的，我们还需要一致化，把修改的结果弄到磁盘里面去，sys_unlink才算完成。
### fstat
   fstat(fd:i32,st:*mut Stat) -> i32,功能是获取文件的状态，参数为fd文件描述符，st文件状态结构体。stat获取起来很简单，主要还是一个地址转化的过程，获取stat的关键主要是参数inodo文件所在inode编号ino，文件类型mode，硬链接数量nlink。
   
   
   
   
   
   
