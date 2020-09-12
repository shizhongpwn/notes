# Ucore-lab1-bootloader启动ucore

### 启动顺序

![image-20200808133439994](Ucore-lab1-bootloader启动ucore.assets/image-20200808133439994.png)

![image-20200808153226726](Ucore-lab1-bootloader启动ucore.assets/image-20200808153226726.png)

读取bootloader之后对ucore os进行加载。关键在于读懂代码。

### C函数调用的实现

这个，没讲什么，就是一些栈现场保存和参数传递的一些规则，比较简单。

### GCC内联汇编

![image-20200808155752711](Ucore-lab1-bootloader启动ucore.assets/image-20200808155752711.png)

感觉还是intel的内联汇编舒服一点。。。。

### 中断

![image-20200808160416267](Ucore-lab1-bootloader启动ucore.assets/image-20200808160416267.png)

每一种中断都有一个对应的中断号，中断号对应着中断处理程序，这些都由操作系统进行实现。

![image-20200808160949640](Ucore-lab1-bootloader启动ucore.assets/image-20200808160949640.png)

不同的特权级在产生中断的时候存在不同的操作。

![image-20200808161953929](Ucore-lab1-bootloader启动ucore.assets/image-20200808161953929.png)

