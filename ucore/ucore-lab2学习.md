# ucore-lab2学习

> K神：https://github.com/Kiprey/Skr_Learning/blob/master/week9-19/uCore/doc/uCore-2.md

## 物理内存探测

BIOS里面的中断调用可以帮助操作系统完成探测物理内存是如何分布的，那些是可用的，那些是不可用的。`bootasm.S`中新增了一段代码，使用BIOS中断检测物理内存总大小。

* 在讲解该部分代码前，先引入一个结构体

  ```c
  struct e820map {      // 该数据结构保存于物理地址0x8000
      int nr_map;       // map中的元素个数
      struct {
          uint64_t addr;    // 某块内存的起始地址
          uint64_t size;    // 某块内存的大小
          uint32_t type;    // 某块内存的属性。1标识可被使用内存块；2表示保留的内存块，不可映射。
      } __attribute__((packed)) map[E820MAX];
  };
  ```

* 以下是bootasm.S中新增的代码，详细信息均以注释的形式写入代码中。

~~~asm
probe_memory:
    movl $0, 0x8000   # 初始化，向内存地址0x8000，即uCore结构e820map中的成员nr_map中写入0
    xorl %ebx, %ebx   # 初始化%ebx为0，这是int 0x15的其中一个参数
    movw $0x8004, %di # 初始化%di寄存器，使其指向结构e820map中的成员数组map
start_probe:
    movl $0xE820, %eax  # BIOS 0x15中断的子功能编号 %eax == 0xE820
    movl $20, %ecx    # 存放地址范围描述符的内存大小，至少20
    movl $SMAP, %edx  # 签名， %edx == 0x534D4150h("SMAP"字符串的ASCII码)
    int $0x15     # 调用0x15中断
    jnc cont      # 如果该中断执行失败，则CF标志位会置1，此时要通知UCore出错
    movw $12345, 0x8000 # 向结构e820map中的成员nr_map中写入特殊信息，报告当前错误
    jmp finish_probe    # 跳转至结束，不再探测内存
cont:
    addw $20, %di   # 如果中断执行正常，则目标写入地址就向后移动一个位置
    incl 0x8000     # e820::nr_map++
    cmpl $0, %ebx   # 执行中断后，返回的%ebx是原先的%ebx加一。如果%ebx为0，则说明当前内存探测完成
    jnz start_probe
finish_probe:
~~~

