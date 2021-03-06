# Ucore-lab1-基础补充

## 保护模式

主要目标：确保应用程序没办法对操作系统进行破坏。

1. 80386处理器就是通过在实模式下初始化控制寄存器(GDTR,LDTR,IDTR,TR等管理寄存器)和页表。
2. 设置CR0寄存器使得其中的保护模式使能位置位，从而进入到保护模式。
3. 在保护模式下，支持内存分页机制，提供了对虚拟内存的良好支持。保护模式下80386支持多任务，还支持优先级机制，不同的程序可以运行在不同的特权级上。特权级一共分0～3四个级别，操作系统运行在最高的特权级0上，应用程序则运行在比较低的级别上；配合良好的检查机制后，既可以在任务间实现数据的安全共享也可以很好地隔离各个任务。

## 内存架构

三个重要的地址空间概念。**逻辑地址，线性地址，物理地址**。

* 一个计算机系统只有一个物理地址空间。
* 线性地址是8086处理器通过段机制控制下形成的地址空间。
* 在操作系统管理下，每个应用程序有自己独立的一个或者多个内存空间段，每个段有各自的起始地址和长度属性，大小不固定，可以让多个程序之间相互隔离，实现对地址空间的保护。

段机制对大量应用程序分散的使用打大内存的支持比较弱，所以加入了页机制。

上述三种地址的关系如下：

- 分段机制启动、分页机制未启动：逻辑地址--->***段机制处理\***--->线性地址=物理地址
- 分段机制和分页机制都启动：逻辑地址--->***段机制处理\***--->线性地址--->***页机制处理\***--->物理地址

但是一般在32位及其以上的架构时，段机制基本不发挥作用。

## 寄存器

段寄存器：

~~~asm
    CS：代码段(Code Segment)
    DS：数据段(Data Segment)
    ES：附加数据段(Extra Segment)
    SS：堆栈段(Stack Segment)
    FS：附加段
    GS 附加段
~~~

标志位寄存器相关：

~~~asm
CF(Carry Flag)：进位标志位；
    PF(Parity Flag)：奇偶标志位；
    AF(Assistant Flag)：辅助进位标志位；
    ZF(Zero Flag)：零标志位；
    SF(Singal Flag)：符号标志位；
    IF(Interrupt Flag)：中断允许标志位,由CLI，STI两条指令来控制；设置IF位使CPU可识别外部（可屏蔽）中断请求，复位IF位则禁止中断，IF位对不可屏蔽外部中断和故障中断的识别没有任何作用；
    DF(Direction Flag)：向量标志位，由CLD，STD两条指令来控制；
    OF(Overflow Flag)：溢出标志位；
    IOPL(I/O Privilege Level)：I/O特权级字段，它的宽度为2位,它指定了I/O指令的特权级。如果当前的特权级别在数值上小于或等于IOPL，那么I/O指令可执行。否则，将发生一个保护性故障中断；
    NT(Nested Task)：控制中断返回指令IRET，它宽度为1位。若NT=0，则用堆栈中保存的值恢复EFLAGS，CS和EIP从而实现中断返回；若NT=1，则通过任务切换实现中断返回。在ucore中，设置NT为0。
~~~

## Ucore基本数据结构

~~~c
struct pmm_manager {
    // XXX_pmm_manager's name
    const char *name;  
    // initialize internal description&management data structure
    // (free block list, number of free block) of XXX_pmm_manager 
    void (*init)(void); 
    // setup description&management data structcure according to
    // the initial free physical memory space 
    void (*init_memmap)(struct Page *base, size_t n); 
    // allocate >=n pages, depend on the allocation algorithm 
    struct Page *(*alloc_pages)(size_t n);  
    // free >=n pages with "base" addr of Page descriptor structures(memlayout.h)
    void (*free_pages)(struct Page *base, size_t n);   
    // return the number of free pages 
    size_t (*nr_free_pages)(void);                     
    // check the correctness of XXX_pmm_manager
    void (*check)(void);                               
};
~~~

`Ucore`里面的双向链表结构

~~~c
struct list_entry {
    struct list_entry *prev, *next;
};
~~~

它没有包含数据区域，因为你涉及思想在于数据结构包含链表节点。比如`free_area_t`，其头指针定义如下：

~~~c
/* free_area_t - maintains a doubly linked list to record free (unused) pages */
typedef struct {
    list_entry_t free_list;         // the list header
    unsigned int nr_free;           // # of free pages in this free list
} free_area_t;
~~~

空闲快链表节定义：

~~~c
/* *
 * struct Page - Page descriptor structures. Each Page describes one
 * physical page. In kern/mm/pmm.h, you can find lots of useful functions
 * that convert Page to other data types, such as phyical address.
 * */
struct Page {
    atomic_t ref;          // page frame's reference counter
    ……
    list_entry_t page_link;         // free list link
};
~~~

自己画了一遍：

![image-20201203150416494](Ucore-lab1-基础补充.assets/image-20201203150416494.png)

这样我们可以让所有的数据结构共享通用的链表操作函数。

> 这种链表方式带来一个很严重的问题，我们怎么获得该结构体？

`le2page宏`

~~~c
#define le2page(le, member)                 \
to_struct((le), struct Page, member)
~~~

`to_struct宏`

~~~c
/* Return the offset of 'member' relative to the beginning of a struct type */
#define offsetof(type, member)                                      \
((size_t)(&((type *)0)->member))

/* *
 * to_struct - get the struct from a ptr
 * @ptr:    a struct pointer of member
 * @type:   the type of the struct this is embedded in
 * @member: the name of the member within the struct
 * */
#define to_struct(ptr, type, member)                               \
((type *)((char *)(ptr) - offsetof(type, member)))
~~~

采用的是gcc编译器技巧：**先求的数据结构的成员变量在本宿主数据结构里面的偏移量，然后根据变量的地址反过来求得属主数据结构变量的地址。**（学到了）

其中`offsetof`巧妙利用了数据偏移结构：首先将0地址强制"转换"为type数据结构（比如struct Page）的指针，再访问到type数据结构中的member成员（比如page_link）的地址，即是type数据结构中member成员相对于数据结构变量的偏移量。在offsetof宏中，这个member成员的地址（即“&((type *)0)->member)”）实际上就是type数据结构中member成员相对于数据结构变量的偏移量。

那么理解了`offsetof`之后`to_struct`就很清楚了：

to_struct宏正是利用这个不变的偏移量来求得链表数据项的变量地址。接下来再分析一下to_struct宏，可以发现 to_struct宏中用到的ptr变量是链表节点的地址，把它减去offsetof宏所获得的数据结构内偏移量，即就得到了包含链表节点的属主数据结构的变量的地址。













