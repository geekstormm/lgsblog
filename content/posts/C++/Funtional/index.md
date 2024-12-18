---
title: C++中的std::function
date: 2024-12-17T23:02:18+08:00
tags:
  - C++
  - functional
categories:
  - C++
---

## 引言

### std::function是什么
std::function 是 C++11 引入的通用可调用对象包装器，用于封装函数、函数对象、Lambda 表达式以及 std::bind 返回的对象等各种可调用类型。它提供了统一的接口，能够在运行时存储和调用这些对象。
本质上就是一个模板类，它主要用来存储和调用可调用对象（通函数，lambda，重载了operator()的类，类的成员函数指针，std::bind的结果）。

### 为什么需要 std::function？
- 传统函数指针的局限性：只能指向函数，无法直接适配仿函数或 Lambda 表达式。
- 多态性的需求：不同类型的可调用对象可以被统一存储和调用。
- 代码灵活性：可以作为参数传递、存储回调函数、实现事件驱动等机制。
典型的适用场景有：
- 回调函数机制（如事件触发、异步任务）
- 函数作为参数传递
- 组合不同可调用对象的接口（如策略模式）

## std::function基本用法

1. 包装普通函数
```C++
#include <functional>
#include <iostream>

void greet(int age) {
    std::cout << "Hello, your age is " << age << std::endl;
}

int main() {
    std::function<void(int)> func = greet;  // 包装普通函数
    func(25);  // 调用函数
    return 0;
}
```
2. 包装 Lambda 表达式
std::function 支持 Lambda 表达式，非常适合简洁的逻辑封装。
```C++
#include <functional>
#include <iostream>

int main() {
    auto lambda = [](int x) { std::cout << "Lambda says: " << x << std::endl; };
    std::function<void(int)> func = lambda;  // 包装 Lambda
    func(42); // 调用

    return 0;
}
```
3. 包装仿函数（函数对象）
函数对象是通过重载 operator() 的类或结构体，可以像函数一样被调用。
```C++
#include <functional>
#include <iostream>

struct Functor {
    void operator()(int x) const {
        std::cout << "Functor called with x = " << x << std::endl;
    }
};

int main() {
    Functor functor;
    std::function<void(int)> func = functor;  // 包装仿函数
    func(100);

    return 0;
}
```
4. 包装成员函数与 std::bind
成员函数不能直接通过 std::function 绑定，需要借助 std::bind。
```C++
#include <functional>
#include <iostream>

class MyClass {
public:
    void show(int x) {
        std::cout << "MyClass::show called with x = " << x << std::endl;
    }
};

int main() {
    MyClass obj;
    auto bound_func = std::bind(&MyClass::show, obj, std::placeholders::_1);
    std::function<void(int)> func = bound_func;  // 包装成员函数

    func(200);  // 调用 MyClass::show(200)
    return 0;
}
```

## std::function与std::bind
std::bind返回的结果是一个可调用对象，并可以将参数进行绑定
语法：
```C++
std::bind(function, arg1, arg2, ..., std::placeholders::_1);
```
- std::placeholders::_1 表示占位符，调用时由实际参数替代。
- 适合部分参数固定的场景。

## std::function的性能与开销
### std::function 的内部实现

std::function 使用了一种叫 类型擦除（Type Erasure） 的技术来适配不同的可调用对象类型。

**类型擦除的原理：**
std::function 内部会存储一个指向基类的指针，并通过多态实现对可调用对象的统一管理。
它会将不同类型的可调用对象（普通函数、Lambda、仿函数等）转换为一个内部通用接口。
虽然这种机制提供了高度灵活性，但会带来一定的 运行时开销。

### std::function 的开销与代价
动态内存分配：对于较大的可调用对象（如捕获很多变量的 Lambda），std::function 可能会动态分配内存。
额外间接调用：由于多态和类型擦除机制，调用封装的函数时会增加一次间接调用的开销。

### 避免开销的方法
在性能敏感的场景中，可以考虑以下替代方案：
- 直接使用模板函数：模板参数在编译时确定类型，零开销。
- 使用 auto：C++14 及以上版本中，auto 类型推导能直接保存可调用对象。
- 使用 std::invoke：C++17 引入的 std::invoke 是一种轻量级的可调用对象调用器。

## 常见应用场景
这里只说我在工作中常实践的几个场景
### 回调函数
实际中一般是通过 `<string, callback>` 的方式进行注册，在此处为简单起见并没有使用。
```C++
#include <functional>
#include <iostream>

void registerCallback(std::function<void()> callback) {
    std::cout << "Triggering callback..." << std::endl;
    callback();
}

int main() {
    registerCallback([]() { std::cout << "Callback executed!" << std::endl; });
    return 0;
}
```

### 事件驱动编程
在事件驱动编程中，可以用 std::function 存储事件处理器。
```C++
#include <functional>
#include <iostream>
#include <map>

class EventManager {
    std::map<std::string, std::function<void()>> events;

public:
    void registerEvent(const std::string& name, std::function<void()> handler) {
        events[name] = handler;
    }

    void triggerEvent(const std::string& name) {
        if (events.count(name)) events[name]();
    }
};

int main() {
    EventManager manager;
    manager.registerEvent("onStart", []() { std::cout << "Start event triggered!" << std::endl; });
    manager.registerEvent("onEnd", []() { std::cout << "End event triggered!" << std::endl; });

    manager.triggerEvent("onStart");
    manager.triggerEvent("onEnd");
    return 0;
}
```

### 策略模式
通过 std::function，可以动态设置逻辑。

```C++
#include <functional>
#include <iostream>

void algorithmA() { std::cout << "Using Algorithm A" << std::endl; }
void algorithmB() { std::cout << "Using Algorithm B" << std::endl; }

class Context {
    std::function<void()> strategy;

public:
    void setStrategy(std::function<void()> func) { strategy = func; }
    void execute() { strategy(); }
};

int main() {
    Context context;

    context.setStrategy(algorithmA);
    context.execute();

    context.setStrategy(algorithmB);
    context.execute();

    return 0;
}
```

### 多线程中的任务分发
结合 std::function 和线程库，任务可以灵活地分发给不同的线程。

```C++
#include <functional>
#include <iostream>
#include <thread>

void task(int id) {
    std::cout << "Task " << id << " is running on thread " << std::this_thread::get_id() << std::endl;
}

int main() {
    std::function<void(int)> func = task;

    std::thread t1(func, 1);
    std::thread t2(func, 2);

    t1.join();
    t2.join();

    return 0;
}
```

## std::function 与 std::move
std::function是一个模板类，既然是类就必然有开销。C++11支持移动语义，可以使用 std::move 避免不必要的复制。
```C++
#include <functional>
#include <iostream>

int main() {
    auto lambda = [](int x) { std::cout << "Lambda executed with " << x << std::endl; };

    std::function<void(int)> func = std::move(lambda);  // 移动 lambda
    func(10);

    return 0;
}
```

## std::function 的替代品
- 如果只需要封装普通函数，可以直接使用函数指针， 更加轻量
- 当类型已知或固定时，使用模板函数或 auto 可以避免 std::function 的动态分配和类型擦除开销。
- C++17的std::invoke

## 总结

### std::function 的优缺点概述
优点：
- 灵活性高，支持普通函数、Lambda、仿函数、std::bind 对象等。
- 接口统一，易于组合和扩展。
- 适合需要动态行为的场景。

缺点：
- 动态分配带来的性能开销。
- 类型擦除机制导致间接调用，影响效率。
- 使用不当可能增加复杂度。

### 使用场景

- 需要存储多种类型的可调用对象。
- 动态注册和调用回调函数。
- 实现灵活的策略模式或事件机制。
