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





> https://dere.press/glibc_tls/
>
> http://hmarco.org/bugs/glibc_ptr_mangle_weakness.html



