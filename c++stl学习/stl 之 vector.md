# stl 之 vector

~~~c++
#include <iostream>
#include <vector>
int main() {
    std::cout << "Hello, World!" << std::endl;
    std::vector<int> v;
    return 0;
}	
~~~

allocator隐藏在所有容器（包括vector）身后，默默**完成内存配置与释放，对象构造和析构的工作**。

## 功能

- 将底层的`alloc`（**内存分配器**，只分配以字节为单位的原始内存，并构建了自己的内存池）封装起来，只保留以对象大小为单位的内存分配
- 对`alloc`分配出来的地址空间进行`placement new` 构造
- 对自身构造出来的内容进行对应的析构（需要和构造进行配对，比如用 `allocator`构造的对象一般只允许被`allocator`所析构）

