## Content
* GPU内存管理
* GPU通信
---
## GPU 内存管理
---
从用户编程角度而言，在使用存储资源时看到的就是CPU指针和GPU指针

    对GPU内存的使用经历了三个阶段：
        第一个阶段是分离内存管理，GPU上运行的Kernel代码不能直接访问CPU内存，在载入Kernel之前或Kernel执行结束之后必须进行显式的拷贝操作；
        第二个阶段是半分离内存管理，Kernel代码能够直接用指针寻址到整个系统中的内存资源；
        第三个阶段是统一内存管理，CPU还是GPU上的代码都可以使用指针直接访问到系统中的任意内存资源；
        对于用户而言，第三个阶段看起来只是提供了一个语法糖，但在底层硬件实现上两者有着显著的差异

### 1. 分离内存管理（Page-locked Memory & Mapped / Zero Copy Memory）
对于分离内存管理，其又可以分为两种，即锁页内存和零拷贝内存。在最原始的方式下，从主机内存拷贝数据到GPU，首先操作系统会分配一块用于数据中转的临时锁页内存，然后将用户缓冲区中的数据拷贝到锁页内存中，再通过PCIe DMA拷贝到GPU显存中  

GPU要使用CPU的数据是需要通过CudaMemcpy进行显式拷贝操作的，这种方式适合大批量的数据传递。如果只是想更新某个标志位，可以使用零拷贝内存（GPU寄存器堆直接与主机内存交互）。  
将主机内存指针进行映射后，Kernel就可以直接使用指针来访问主机内存了，读取的数据会直接写入寄存器中

<br>
  
### 2. 半分离内存管理（Unified Virtual Address UVA）
**`统一虚拟地址空间`**  

<b>语法糖</b> &emsp; 将原有的四个方向的拷贝函数合成了一个，用户调用统一的拷贝函数，由Cuda Runtime来判定数据源和目标所在的物理地址  

对于UVA，GPU访问主存是直接将数据搬到寄存器里的，不经过其显存，这也就意味着每次访问都至少要经过一次PCIe操作

<br>

### 3. 统一内存管理（Unified Address UA）  
**`统一虚拟地址空间`**  

<b>将系统内的所有内存资源都整合到相同的虚拟地址空间中。</b>不管是CPU还是GPU代码，不用再区分其指针指向的空间  
<b>UM本质上还是一种语法糖</b>

实际上，直到Pascal架构才算真正有了对UM的硬件上的支持。在Pascal架构之前，Kepler和Maxwell仅仅还是沿用了前面讲的CPU数据搬移到GPU寄存器中，只是在CUDA Runtime中提供了对地址的判断。而在Pascal架构上，实现了对物理内存页的按需迁移，GPU和CPU的并发访问，内存超额配置等，以及在Volta架构上又进一步实现了访问计数器，GPU和CPU的Cache一致性等新特性。

Pascal架构之前的UM：  
首先分配GPU显存，这时GPU的MMU会分配一段物理内存，然后构造页表项； 当CPU指针访问这段显存时，发生缺页异常，进行物理页迁移，将GPU的物理页迁移到CPU内存中，此时GPU的页表会进行释放； 当Kernel使用该地址时，会再次构造GPU页表项，将CPU内存页迁移到GPU上。
- 不支持 <b>内存超额配置</b>（申请的内存数量不能超过GPU显存总量）
- 不支持 <b>按需页迁移</b> &emsp; 当GPU显存已经塞满时，如果要访问CPU内存，数据会直接进入GPU寄存器，而不会对显存进行置换，如果频繁访问CPU内存就会带来较大的开销
  
在Pascal之后的架构对GPU缺页异常提供了支持，在分配GPU内存时，只是分配了一个页表项，没有进行实际的显存分配（仅分配了虚拟地址）  
当CPU代码访问内存时，发生缺页异常，分配CPU物理内存页，创建CPU页表项。当GPU代码访问时，发生GPU缺页异常，CPU内存页通过PCIe被迁移到GPU显存中（此时进行GPU内存分配）。  

<br>

---
## GPU通信
---
可粗略分为`CPU控制`与`GPU控制`  
第一种情况下，GPU运算完成后，将数据同步给CPU，由CPU执行MPI通信。（由CPU进行通信的控制以及与计算过程的同步，存在PCIe的通信开销）

GPUDirect RDMA中，GPU和网卡可以直接通过PCIe进行数据交互，避免了跨节点通信过程中内存和CPU的参与

    为了实现CPU控制通信，数据进行P2P搬移，要解决：网卡如何直接读写GPU显存（向网卡注册GPU虚拟地址）  
    为了实现GPU直接控制通信，要解决： GPU如何访问通信资源 & GPU如何与网卡进行同步
RDMA通信中，唯一能够且有必要由GPU控制的流程就是`提交工作请求和同步过程`  
提交工作请求涉及通信资源：  
* 按资源类型：上下文资源，队列资源和数据区域
* 按资源位置：位于设备内存和主机内存  

其中，部分资源是仅由网卡进行访问的，这部分资源可以留在主机内存中保持不变，例如队列基地址，通信序列号等。另一部分资源是由CPU进行读写或临时创建的，因此需要详细考虑  