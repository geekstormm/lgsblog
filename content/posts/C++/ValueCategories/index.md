---
title: 深入浅出C++中的左值与右值
date: 2024-12-17T01:14:18+08:00
tags:
  - C++
  - Value Categories
categories:
  - C++
---

## 基本概念

在开始讲解左值和右值之前，咱们需要先理解两个基本概念：类型（Type）和值类别（Value Categories）。这两个概念经常被混淆，但它们其实是完全不同的东西。

### 1.1 Type vs Value Categories

想象你有一个快递包裹：

Type（类型）就像是这个包裹的属性：是个箱子还是个信封，是大是小，是重是轻。

Value Categories（值类别）则像是这个包裹的状态：是在仓库里存放的，还是正在运输中的，或是即将被销毁的。

让我们用简单的代码来说明：

```C++
int x = 42;            // x的类型(Type)是int
int& ref = x;          // ref的类型是int的引用
int&& rref = 42;       // rref的类型是int的右值引用
```

### 1.2 两者的关系

很多初学者会困惑："左值引用"和"右值"这两个概念看起来很像，但为什么有时候左值引用能指向右值？这是因为：

-   "左值引用"是一个类型（Type）
-   "右值"是一个值类别（Value Category）

就像快递包裹可以从仓库（左值）转变为运输状态（右值）一样，一个表达式的值类别也可以改变。

## 左值（lvalue）

### 2.1 什么是左值？

最简单的理解：**左值就是可以取地址的表达式**。想象你的家：

-   你的家有一个固定的地址 -> 这就像左值
-   你可以随时回到这个地址 -> 这就像可以多次访问左值

### 2.2 常见的左值例子：

```C++
int x = 10;          // x是左值
string str = "hello"; // str是左值
int arr[10];         // arr是左值

// 你可以对左值取地址
int* ptr = &x;       // 正确：x是左值
```

## 将亡值（xvalue）

### 3.1 什么是将亡值？

将亡值就像是即将搬家的物品：

-   它现在还在原地（有确定位置）
-   但马上就要被移动到新地方了
-   移动后原来的位置就空了

### 3.2 最常见的将亡值例子：

```C++
string str = "hello";
string new_str = std::move(str);  // str变成了将亡值
// 此时str还存在，但它的内容已经被移动走了
```

## 纯右值（prvalue）

### 4.1 什么是纯右值？

纯右值就像是临时的东西：

-   数字42
-   临时计算的结果
-   函数返回的临时对象

### 4.2 简单例子：

```C++
int x = 42;          // 42是纯右值
int y = x + 1;       // x + 1的结果是纯右值
string getName() {
    return "John";   // "John"是纯右值
}
```

## 为什么需要这些概念？

想象你在操作文件：

如果是把文件从一个目录拷贝到另一个目录（左值到左值）：

-   需要完整复制一份
-   费时费力

如果是把文件直接剪切到新目录（将亡值）：

-   直接移动过去就行
-   更快更省力

下面是一些实际使用案例（没有使用函数返回对象的例子是因为现代C++编译器都有RVO）：

**容器元素转移：**

```C++
class BigData {
    vector<string> data;
public:
    BigData(vector<string>&& vec) : data(std::move(vec)) {} // 移动构造
};

vector<string> vec = {"很长的字符串1", "很长的字符串2", "很长的字符串3"};
BigData bd(std::move(vec));  // 如果不用move，会复制整个vector
```

在容器中重用对象

```C++
vector<string> strings;
string str = "很长的字符串";
strings.push_back(str);             // 复制
strings.push_back(std::move(str));  // 移动，避免复制
// 此时str为空，可以重用
str = "另一个很长的字符串";        // 重用str
```

unique_ptr的转移

```C++
class Resource {
    unique_ptr<BigObject> ptr;
public:
    void setResource(unique_ptr<BigObject>&& r) {
        ptr = std::move(r);  // 必须使用move，unique_ptr不能复制
    }
};

Resource res;
auto obj = make_unique<BigObject>();
res.setResource(std::move(obj));  // 必须move，否则编译错误
```

类成员的高效替换

```C++
class MyClass {
    vector<string> data;
public:
    void swap(MyClass& other) {
        data = std::move(other.data);  // 高效移动而不是复制
        other.data = std::move(data);  // 恢复other的数据
    }
};
```

在STL算法中移动元素

```C++
vector<string> source = {"长字符串1", "长字符串2", "长字符串3"};
vector<string> dest;
// 移动而不是复制元素
std::copy(std::make_move_iterator(source.begin()),
          std::make_move_iterator(source.end()),
          std::back_inserter(dest));
// source中的字符串现在都是空的
```

## 实际应用建议

### 对于初学者：

1. 当你需要转移一个大对象的所有权时，使用std::move：

```C++
string str = "hello";
vector<string> vec;
vec.push_back(std::move(str));  // str的内容被移动到vector中
// 注意：此时str变成了空字符串
```

2. 注意移动后的对象状态：

```C++
string str = "hello";
string new_str = std::move(str);
// 此时str是空的，不要继续使用它
```

3. 使用不可复制但是可以移动的类型如`unique_ptr`时。
4. 不要对const对象使用'std::move'。

## 总结

-   类型（Type）和值类别（Value Categories）是不同的概念
-   左值是可以取地址的表达式
-   将亡值是可以被移动的对象
-   纯右值是临时的值

这些概念初看可能很复杂，但随着你的C++编程经验增加，它们会变得越来越清晰。不要期望一次就完全理解，这是一个渐进的过程。
