# IDApro-Rebuilding virtual function table (vftable)

### 前置操作

**关键图片**

![image-20200806104447637](IDApro-Rebuilding virtual function table (vftable).assets/image-20200806104447637.png)

定义了一个类 B，一个派生类D ,可以看到里面定义了一些虚函数（关键字为virtual）。代码如下

~~~c++
#include<iostream>
using namespace std;
class base {
public:
	virtual void print()
	{
		cout << "print base class!" << endl;
	}
	virtual void print1()
	{
		cout << "print1 base class!" << endl;
	}
	void show()
	{
		cout << "show base class!" << endl;
	}
};
class derived :public base {
public:
	void print()
	{
		cout << "print derived class" << endl;
	}
	void show()
	{
		cout << "show derived class" << endl;
	}
};
int main()
{
	base* bptr;
	derived d;
	bptr = &d;
	bptr->print();
	bptr->print1();
	bptr->show();
	return 0;
}
~~~

编译之后拖入idapro。

### 插件

https://github.com/0xgalz/Virtuailor

载入

![image-20200806113739696](IDApro-Rebuilding virtual function table (vftable).assets/image-20200806113739696.png)

如果报错的话，记得把main.py的48，49行改成如下的样子。

![image-20200806113650667](IDApro-Rebuilding virtual function table (vftable).assets/image-20200806113650667.png)

打开如下：

![image-20200806114058969](IDApro-Rebuilding virtual function table (vftable).assets/image-20200806114058969.png)

我们可以更改起始地址和结束地址来进行扫描

![image-20200806114136027](IDApro-Rebuilding virtual function table (vftable).assets/image-20200806114136027.png)

可以再输出窗口看到过程：

![image-20200806114338315](IDApro-Rebuilding virtual function table (vftable).assets/image-20200806114338315.png)

但是我们可以看到没什么变化，因为我们要用debug 调试器执行一遍，所以我们可以按照introduction里面的过程选择调试器（mac党哭晕在厕所，在考虑配置一下win10虚拟机环境了），运行之后就会在structures里面看到变化：

![image-20200806114642163](IDApro-Rebuilding virtual function table (vftable).assets/image-20200806114642163.png)

通过;后面的注释，我们可以通过双击很方便的找到程序哪里引用了这些函数，和函数名，然后在idapro左侧的function name栏里面搜索这个函数，然后点击中间框里面的函数名

![image-20200806115002984](IDApro-Rebuilding virtual function table (vftable).assets/image-20200806115002984.png)

，通过快捷键x的交叉引用，我们可以很快的得出整个程序哪里引用了这个函数。







