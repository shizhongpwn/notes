## ucore的下载

有多种方式可以进行实验

一：

通过清华的在线实验楼，不过如果想要保存环境的话需要会员（money）

https://www.shiyanlou.com/courses/221

（不用配环境的快乐你无法想象）

二：

网上clone ubuntu环境

https://github.com/kiukotsu/ucore （2015版）

https://github.com/chyyuu/os_kernel_lab/tree/master （2020版,对lab2部分进行了修改,具体可以查看 /kern/mm/default_pmm.c）

2020版本下载完后发现没有labcodes，别急，那是在另一个分支上，此时进入你下载好的文件夹内，依次输入

```shell
git checkout master
git pull origin master
```

就ok了。

## 整体实验框架

![image-20200803130512967](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/image-20200803130512967.png)

**LAB 0**

主要是实验环境的配置，照着配就行

**LAB 1**

![image-20200803130741258](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/image-20200803130741258.png)

**LAB 2**

![image-20200803130948148](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/image-20200803130948148.png)

**LAB 3**

![image-20200803130958930](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/image-20200803130958930.png)

**LAB 4**

![image-20200803131015161](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/image-20200803131015161.png)

**LAB 5**

![image-20200803131027641](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/image-20200803131027641.png)

**LAB 6**

![image-20200803131041778](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/image-20200803131041778.png)

**LAB 7**

![image-20200803131056545](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/image-20200803131056545.png)

**LAB 8**

![image-20200803131109230](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/image-20200803131109230.png)

## 相关的学习步骤

+ 视频过一遍
  - https://www.bilibili.com/video/BV1w7411P7qk?p=11
+ 看实验手册
  - https://chyyuu.gitbooks.io/ucore_os_docs/content/lab5/lab5_2_1_exercises.html
+ 对着代码撸，想清楚如何实现视频中相关理论的
+ 在手册和代码间反复横跳

## LAB 1

总体来说，坑被踩的差不多了，踩坑完毕后的具体链接:

[https://github.com/tina2114/Sakura_University/tree/master/%E7%AC%AC%E5%8D%81%E4%B8%80%E5%91%A8/LAB%201](https://github.com/tina2114/Sakura_University/tree/master/第十一周/LAB 1)

## LAB 2

此处应该重点理解相应数据结构的实现，主要就集中在init，alloc，free三个函数的实现上，这里写错了，或者理解不到位，后面的LAB，相信我，会错的怀疑人生

同样，坑被踩的差不多了，踩坑完毕后的具体链接:

[https://github.com/tina2114/Sakura_University/tree/master/%E7%AC%AC%E5%8D%81%E4%B8%80%E5%91%A8/LAB%202](https://github.com/tina2114/Sakura_University/tree/master/第十一周/LAB 2)

## LAB 3

在LAB 2的基础上，重点得实现FIFO

同样，坑被踩的差不多了，踩坑完毕后的具体链接:

[https://github.com/tina2114/Sakura_University/tree/master/%E7%AC%AC%E5%8D%81%E4%B8%80%E8%87%B3%E5%8D%81%E4%BA%8C%E5%91%A8/LAB%203](https://github.com/tina2114/Sakura_University/tree/master/第十一至十二周/LAB 3)

## LAB 4

实现内核线程，好像没什么坑？

[https://github.com/tina2114/Sakura_University/tree/master/%E7%AC%AC%E5%8D%81%E4%B8%80%E8%87%B3%E5%8D%81%E4%BA%8C%E5%91%A8/LAB%204](https://github.com/tina2114/Sakura_University/tree/master/第十一至十二周/LAB 4)

## LAB 5

**坑王**

**首先**，练习0很重要，看清楚，人家要求你在lab 1,2,3,4写完的代码上对其改动，使其适应lab5的用户线程

需要改动的有trap函数，alloc_proc函数，do_fork函数，trap_dispatch函数（有些人需要），甚至还有/kern/init文件（这也是有些人需要）

**其次**，正常的练习部分先略过不说，不是坑的重点。你写完练习后，make grade进行打分的时候，如果你的tools/grade.sh的文件是labcodes_answer的，你会发现check_out是失败的，它缺少了check_slab()函数的完善。这是因为，answer中的grade把前面所有lab的拓展练习也一并检测了，但是坑的地方是，answer本身并没有实现这些拓展，所以分数总是跑不满。

但是，你自己的labcodes的grade.sh是没有加入对拓展练习的检测的。

总结一下就是，你写了拓展，用answer的grade.sh检测，没写就用原本的grade.sh检测，别乱覆盖



**再而**，在你make qemu的时候，会出现两种状况

![img](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/186BF41E99DFF58887671483DCC4CF86.jpg)

以及

![image-20200816212744968](https://gitee.com/zhzzhz/blog_warehouse/raw/master/img/image-20200816212744968.png)


这两种都是对的，第二种你多跑几次，就变成了第一种（我也不知道为什么这检测文件会发生变动）
## LAB6
lab6在proc.c中init_main函数 有一个对于 assert(nr_free_pages_store == nr_free_pages()); 此处还未找到bug,应该是调度器原因
无论是RR调度器 还是 answer的stride调度器 都有问题 而lab5就没出现问题 表示fork那部分应该是没有问题的
但是这里 answer是有这句话,而要做的lab那里 这段是不存在的,所以answer就算替换成实验的grade.sh,依然拿不到满分,除非把这句注释掉,等我后面找到bug再回来补充

从lab5 开始 可以单独跑user中的进程,即 make run-functionname,如 make run-forktest,也是从这个函数开始 nr_free_pages()返回的值就开始缺少页数量了
当然 也可能是没写slab的原因 因为追溯到相应函数 发现了关键词slab
## Continue
