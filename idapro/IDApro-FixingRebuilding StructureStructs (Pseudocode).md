#IDApro-Fixing/Rebuilding Structure/Structs (Pseudocode)

### F5

ida的F5来源于一个功能强大的插件，可以把汇编代码转换为伪C代码来方便我们的逆向分析。

汇编代码：

![image-20200806100209759](IDApro-FixingRebuilding StructureStructs (Pseudocode).assets/image-20200806100209759.png)

F5之后：

![image-20200806100141628](IDApro-FixingRebuilding StructureStructs (Pseudocode).assets/image-20200806100141628.png)

通过变量重命名(快捷键N)之后我们几乎可以看到跟c差不多的代码结构。

### Structures功能

经过对f5伪代码的分析之后，我们可以辨认出一个数据结构，为了在伪代码里面更清楚的观看这个数据结构，我们需要在Structures功能区创建一个我们需要的数据结构。

![image-20200806100819170](IDApro-FixingRebuilding StructureStructs (Pseudocode).assets/image-20200806100819170.png)

这个区域告诉了我们改如何进行操作。

Tips:mac上的insert键 字母i

![image-20200806101442415](IDApro-FixingRebuilding StructureStructs (Pseudocode).assets/image-20200806101442415.png)

这样就进行了结构体创建，快捷键d可以为结构体增加新成员。

![image-20200806101727732](IDApro-FixingRebuilding StructureStructs (Pseudocode).assets/image-20200806101727732.png)

鼠标点击db这个类型，然后按d可以进行类型切换，db一字节,dw二字节,dd四字节,dq八字节。

鼠标电子field_0然后快捷键n可以重命名：

![image-20200806102036092](IDApro-FixingRebuilding StructureStructs (Pseudocode).assets/image-20200806102036092.png)

同时若想继续添加结构体成员，我们可以按d，不过光标要放在最下面的ends哪里。

以此类推我们可以创建我们的结构体，如何应用到伪代码里面？

#### 结构体应用

![image-20200806102404995](IDApro-FixingRebuilding StructureStructs (Pseudocode).assets/image-20200806102404995.png)

鼠标点击你想更改的变量，然后快捷键y,填入你的结构体名称就可以了，或者可以![image-20200806102853560](IDApro-FixingRebuilding StructureStructs (Pseudocode).assets/image-20200806102853560.png)

来选择已有的结构体类型，更新之后如下。

![image-20200806102945798](IDApro-FixingRebuilding StructureStructs (Pseudocode).assets/image-20200806102945798.png)

如果之后你由更改了之前写好的结构体类型，记得再次在伪代码里面导入一次。