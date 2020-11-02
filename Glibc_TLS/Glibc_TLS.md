# Glibc_TLS

> Pointer_guard机制的原理以及绕过。参考：

TLS(Thread Local Storage，线程本地存储)，我们可以使用`__thread`关键字来告知编译器一个变量应该放入TLS。

**TLS只能被当前线程访问和修改**，所以其必然不能和普通变量一样存储到某个段里面（bss,data之类）。

Example:

~~~c
#include <pthread.h>
#include <stdio.h>
 
static __thread int a = 12345; // 0x3039
__thread unsigned long long b = 56789; // 0xddd5
__thread int c;
 
void try(void *tmp) {
    printf("try: a = %lx, b = %llx, c = %s\n", a, b, &c);
    return;
}
 
int main(void) {
    a = 0xdeadbeef;
    b = 0xbadcaffe;
    c = 0x61616161;
    printf("main thread: a = %lx, b = %llx, c = %s\n", a, b, &c);
    pthread_t pid;
    pthread_create(&pid, NULL, try, NULL);
    pthread_join(pid, NULL);
    return 0;
}

~~~

> 编译指令：gcc tls.c -pthread -g -o tls
>
> Pthread_create : https://blog.csdn.net/liangxanhai/article/details/7767430
>
> pthread_join()函数，以阻塞的方式等待thread指定的线程结束。当函数返回时，被等待线程的资源被收回。如果线程已经结束，那么该函数会立即返回。并且thread指定的线程必须是joinable的。

结果输出：

~~~c
main thread: a = deadbeef, b = badcaffe, c = aaaa
try: a = 3039, b = ddd5, c = 
~~~

可以看到，尽管都是全局变量，但是线程之间并没有互相影响。

gdb加载，我们用`info files`可以查看文件结构：

~~~
Entry point: 0x5555555546b0
	0x0000555555554270 - 0x000055555555428c is .interp
	0x000055555555428c - 0x00005555555542ac is .note.ABI-tag
	0x00005555555542ac - 0x00005555555542d0 is .note.gnu.build-id
	0x00005555555542d0 - 0x00005555555542ec is .gnu.hash
	0x00005555555542f0 - 0x00005555555543e0 is .dynsym
	0x00005555555543e0 - 0x00005555555544ab is .dynstr
	0x00005555555544ac - 0x00005555555544c0 is .gnu.version
	0x00005555555544c0 - 0x0000555555554510 is .gnu.version_r
	0x0000555555554510 - 0x00005555555545d0 is .rela.dyn
	0x00005555555545d0 - 0x0000555555554630 is .rela.plt
	0x0000555555554630 - 0x0000555555554647 is .init
	0x0000555555554650 - 0x00005555555546a0 is .plt
	0x00005555555546a0 - 0x00005555555546a8 is .plt.got
	0x00005555555546b0 - 0x0000555555554932 is .text
	0x0000555555554934 - 0x000055555555493d is .fini
	0x0000555555554940 - 0x0000555555554991 is .rodata
	0x0000555555554994 - 0x00005555555549d8 is .eh_frame_hdr
	0x00005555555549d8 - 0x0000555555554b00 is .eh_frame
	0x0000555555754d80 - 0x0000555555754d90 is .tdata
	0x0000555555754d90 - 0x0000555555754d94 is .tbss
	0x0000555555754d90 - 0x0000555555754d98 is .init_array
	0x0000555555754d98 - 0x0000555555754da0 is .fini_array
	0x0000555555754da0 - 0x0000555555754fa0 is .dynamic
	0x0000555555754fa0 - 0x0000555555755000 is .got
	0x0000555555755000 - 0x0000555555755010 is .data
	0x0000555555755010 - 0x0000555555755018 is .bss
~~~

可以看到sections里面多了`tdata`和`tbss`与正常的data以及bss section相似，两者分别存储已经初始化的线程变量以及未初始化的线程变量。

~~~c
pwndbg> telescope 0x0000555555754d80
00:0000│   0x555555754d80 ◂— 0x3039 /* '90' */
01:0008│   0x555555754d88 ◂— 0xddd5
02:0010│   0x555555754d90 (__init_array_start) —▸ 0x5555555547b0 (frame_dummy) ◂— push   rbp
03:0018│   0x555555754d98 (__do_global_dtors_aux_fini_array_entry) —▸ 0x555555554770 (__do_global_dtors_aux) ◂— cmp    byte ptr [rip + 0x200899], 0
04:0020│   0x555555754da0 (_DYNAMIC) ◂— 0x1
... ↓
07:0038│   0x555555754db8 (_DYNAMIC+24) ◂— 0x63 /* 'c' */
~~~

可以看到，tdata段里面存储的确实是我们设定的值。

反汇编try函数：

~~~asm
pwndbg> disassemble try
Dump of assembler code for function try:
   0x00005555555547ba <+0>:	push   rbp
   0x00005555555547bb <+1>:	mov    rbp,rsp
   0x00005555555547be <+4>:	sub    rsp,0x10
   0x00005555555547c2 <+8>:	mov    QWORD PTR [rbp-0x8],rdi
   0x00005555555547c6 <+12>:	mov    rdx,QWORD PTR fs:0xfffffffffffffff0
   0x00005555555547cf <+21>:	mov    eax,DWORD PTR fs:0xffffffffffffffe8
   0x00005555555547d7 <+29>:	mov    rcx,QWORD PTR fs:0x0
   0x00005555555547e0 <+38>:	add    rcx,0xfffffffffffffff8
   0x00005555555547e7 <+45>:	mov    esi,eax
   0x00005555555547e9 <+47>:	lea    rdi,[rip+0x158]        # 0x555555554948
   0x00005555555547f0 <+54>:	mov    eax,0x0
   0x00005555555547f5 <+59>:	call   0x555555554680 <printf@plt>
   0x00005555555547fa <+64>:	nop
   0x00005555555547fb <+65>:	leave  
   0x00005555555547fc <+66>:	ret    
End of assembler dump.
~~~

![image-20201023182625264](Glibc_TLS.assets/image-20201023182625264.png)

我们单步调试结果如下：

但是fs寄存器显示为0，同时直接访问改地址报错，其原因是[gdb没有权限访问](https://stackoverflow.com/questions/10354063/how-to-use-a-logical-address-in-gdb)fs寄存器，但是`fsbase`指令可以访问：

~~~
pwndbg> fsbase 
0x7ffff77c4700
~~~

我们vmmap查看可以发现，这其实是指向了一段开辟的内存空间。

## X86_64_ABI要求的TLS结构

TLS（Thread Local Storage）的结构与TCB（Thread Control Block）以及dtv（dynamic thread vector）密切相关，每一个线程中每一个使用了TLS功能的module都拥有一个TLS Block。这几者的关系如下图所示1：

![image-20201023193502783](Glibc_TLS.assets/image-20201023193502783.png)

`TLS Blocks`分为两类，一类是装载程序的时候已经存在的（位于TCB前），这部分是`static TLS`，右边的Blocks是动态分配的，他们被使用dlopen函数在程序运行的时候动态装载的模块使用。

`TCB`作为线程控制块，保存`dtv`数组的入口（看图箭头），然后`dtv`数组里面的每一项都对应着`TLS Block`的入口，其值为指针，指向`TLS Block`数据快。

> `dtv`数组的第一个成员是一个计数器，每当程序使用dlopen函数或者dlfree函数记载一个具备TLS变量的module的时候，改计数器的值都会加一，从而保证了程序内版本的一致性。（图中gen）

同时elf文件本身对应的`TLS Block`一定在`dtv`数组里面占据索引为1的位置，**且位置与`TCB`相邻**。

图中的tp1指针（TCB上面）是因为，在i386架构上，这个指针为gs寄存器，x86_64架构上其为fs寄存器，因为该指针和ELF文件本身对应的TLS block之间的偏移是固定的，所以程序在编译的时候就可以将ELF中线程变量的地址硬编码到目标文件里面。

# Glibc-TLS具体实现

多线程情况下（CTF_TLS一般就考得多线程）：

> [pthread_create.c](https://code.woboq.org/userspace/glibc/nptl/pthread_create.c.html)的源码，我调试使用的是glibc 2..27

非主线程：

* pthread_create.c的源码还是挺复杂的，但是我们边调试边看

![image-20201102153208466](Glibc_TLS.assets/image-20201102153208466.png)

从这里开始，执行流从pthread_create.c跳转到了allocatestack.c

~~~c
int err = ALLOCATE_STACK (iattr, &pd); //pthread.c里面的新栈分配
~~~

![image-20201102153328398](Glibc_TLS.assets/image-20201102153328398.png)

这是一条`pthread_create函数`跳转到[allocate_stack](https://code.woboq.org/userspace/glibc/nptl/allocatestack.c.html#allocate_stack)函数的调用链,这个函数的注释如下：

~~~c

/* Get a stack frame from the cache.  We have to match by size since
   some blocks might be too small or far too large.  */
~~~

可以看到肯定是要进行空间分配了。

分配方式：通过调用函数为新线程分配栈

![image-20201102201237056](Glibc_TLS.assets/image-20201102201237056.png)

可以看接下来的代码：

~~~c
 pd = (struct pthread *) ((((uintptr_t) mem + size)
                                    - TLS_TCB_SIZE)
                                   & ~__static_tls_align_m1);
~~~

通过这个我们就知道，在分配的空间的底部形成了一个数据结构：

![image-20201102211909390](Glibc_TLS.assets/image-20201102211909390.png)

~~~c
/* 进行这一步操作后内存布局如下：
 * 
 *                                  TLS_TCB_SIZE
 *                                        ^
 *                            +-----------+----------+
 *                            |                      |
 * ---------------------+----------------------------+
 *                      |     |                      |
 *                      | pad |                      |
 *                      |     |                      |
 * ---------------------+----------------------------+
 *                            ^                      ^
 *                            +                      +
 *                           pd                mmap area end
 *
 */
~~~

> 从左向右看，新栈的底部被分配了一个容纳pd结构体的空间，该结构体的类型为struct pthread，我们称其为一个thread descriptor，该结构体的第一个域为tchhead_t类型，其定义如下：

~~~c
typedef struct
{
  void *tcb;        /* Pointer to the TCB.  Not necessarily the
               thread descriptor used by libpthread.  */
  dtv_t *dtv;
  void *self;       /* Pointer to the thread descriptor.  */
  int multiple_threads;
  int gscope_flag;
  uintptr_t sysinfo;
  uintptr_t stack_guard;
  uintptr_t pointer_guard;
  unsigned long int vgetcpu_cache[2];
  /* Bit 0: X86_FEATURE_1_IBT.
     Bit 1: X86_FEATURE_1_SHSTK.
   */
  unsigned int feature_1;
  int __glibc_unused1;
  /* Reservation of some values for the TM ABI.  */
  void *__private_tm[4];
  /* GCC split stack support.  */
  void *__private_ss;
  /* The lowest address of shadow stack,  */
  unsigned long long int ssp_base;
  /* Must be kept even if it is no longer used by glibc since programs,
     like AddressSanitizer, depend on the size of tcbhead_t.  */
  __128bits __glibc_unused2[8][4] __attribute__ ((aligned (32)));
 
  void *__padding[8];
} tcbhead_t;

~~~

到此为止我们就可以知道，pthread会分配一个新栈给线程，然后在新栈的底部放入一个`pthread`数据结构。

### 父线程TCB的继承

上述栈分配完成后，在源码里先在数据结构里面填写了一些内容，

![image-20201102214748179](Glibc_TLS.assets/image-20201102214748179.png)

~~~c
/* Copy the stack guard canary.  */
#ifdef THREAD_COPY_STACK_GUARD
  THREAD_COPY_STACK_GUARD (pd);
#endif
  /* Copy the pointer guard value.  */
#ifdef THREAD_COPY_POINTER_GUARD
  THREAD_COPY_POINTER_GUARD (pd);
#endif
~~~

接下来就是将父进程的canary复制到当前进程的TCB结构体里面，事实上，在fs寄存器没有被改变之前，其中存放着父进程TCB的地址，可以使用`THREAD_SELF`宏来获取父线程的TCB指针。

> canary是存放在pthread结构体里面的，别搞混了，可以发现在执行的前后线程pthread结构里面的`stack_guard`变成了和父进程一样。







> https://dere.press/glibc_tls/
>
> http://hmarco.org/bugs/glibc_ptr_mangle_weakness.html



