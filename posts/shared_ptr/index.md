# C&#43;&#43; Shared_ptr 循环引用


# 循环引用会导致什么

正常的对象在生命周期结束后应该就释放持有的资源了, 但是如果两个shared_ptr对象互相指向对方, 在生命周期结束后两个变量确实会被销毁, 但是他们两个持有的资源依然会在内存中, 此时就引起了内存泄漏.

# 典型的循环引用场景

首先来看一个典型的循环引用场景: 

```C&#43;&#43;
class Node {
public:
    std::shared_ptr&lt;Node&gt; next;  // 指向下一个节点
    std::shared_ptr&lt;Node&gt; prev;  // 指向前一个节点
    int data;
    
    Node(int d) : data(d) {
        std::cout &lt;&lt; &#34;Node &#34; &lt;&lt; data &lt;&lt; &#34; created\n&#34;;
    }
    
    ~Node() {
        std::cout &lt;&lt; &#34;Node &#34; &lt;&lt; data &lt;&lt; &#34; destroyed\n&#34;;
    }
};

void createNodes() {
    auto node1 = std::make_shared&lt;Node&gt;(1);
    auto node2 = std::make_shared&lt;Node&gt;(2);
    
    // 创建循环引用
    node1-&gt;next = node2;
    node2-&gt;prev = node1;
    
    // 函数结束时，即使局部变量node1和node2被销毁
    // 由于循环引用，这些Node对象也不会被删除
}
```

如上所示, 当调用完createNodes后, 按理来说两个临时对象都应该被销毁, 但实际上两个对象被销毁后, Node对象依然存在, 因为两个Node的引用计数不为0.

```C&#43;&#43;
int main() {
    createNodes();
    // 你会看到Node被创建的输出
    // 但不会看到Node被销毁的输出
    // 这表明发生了内存泄漏
    
    std::cout &lt;&lt; &#34;Program ending\n&#34;;
}
```

# 观察者模式中的循环引用

来看代码, 在观察者模式中, 如果观察者和被观察者互相持有对方的引用(观察者需要知道观察对象的状态, 观察对象想知道被几个对象观察), 那么就会发生循环引用.

```C&#43;&#43;
// 观察者模式中常见的循环引用
class Subject;
class Observer {
public:
    std::shared_ptr&lt;Subject&gt; subject;  // 持有Subject的引用
    virtual ~Observer() {
        std::cout &lt;&lt; &#34;Observer destroyed\n&#34;;
    }
};

class Subject {
public:
    std::vector&lt;std::shared_ptr&lt;Observer&gt;&gt; observers;  // 持有Observer的引用
    virtual ~Subject() {
        std::cout &lt;&lt; &#34;Subject destroyed\n&#34;;
    }
};

void test() {
    auto subject = std::make_shared&lt;Subject&gt;();
    auto observer = std::make_shared&lt;Observer&gt;();
    
    // 建立循环引用
    subject-&gt;observers.push_back(observer);
    observer-&gt;subject = subject;
    
    // 函数结束时，两个对象都不会被销毁
}
```

# 如何检测循环引用

1.  直接通过调试或者打日志的方式来观察引用计数：

```C&#43;&#43;
LOGI(ptr.use_count());
```

如果发现对象的引用计数一直不为0，就可能存在循环引用。

1.  使用内存泄漏工具

老生常谈了，使用Valgrind，ASAN等工具来进行排查

1.  Code Review
    1.  检查类之间依赖关系
    2.  检查互相持有shared_ptr的类

# 循环引用的解决方案

## Node循环引用的解决方案

```C&#43;&#43;
class Node {
public:
    std::shared_ptr&lt;Node&gt; next;   // 强引用
    std::weak_ptr&lt;Node&gt; prev;     // 弱引用
    int data;
    
    Node(int d) : data(d) {
        std::cout &lt;&lt; &#34;Node &#34; &lt;&lt; data &lt;&lt; &#34; created\n&#34;;
    }
    
    ~Node() {
        std::cout &lt;&lt; &#34;Node &#34; &lt;&lt; data &lt;&lt; &#34; destroyed\n&#34;;
    }
    
    void process() {
        // 使用weak_ptr时需要先lock
        if (auto p = prev.lock()) {
            std::cout &lt;&lt; &#34;Previous node data: &#34; &lt;&lt; p-&gt;data &lt;&lt; &#34;\n&#34;;
        }
    }
};
```

## 观察者模式循环引用的正确实现

```C&#43;&#43;
class Subject;
class Observer {
public:
    std::weak_ptr&lt;Subject&gt; subject;  // 使用weak_ptr
    virtual ~Observer() {
        std::cout &lt;&lt; &#34;Observer destroyed\n&#34;;
    }
};

class Subject {
public:
    std::vector&lt;std::weak_ptr&lt;Observer&gt;&gt; observers;  // 使用weak_ptr
    
    void notify() {
        // 清理失效的观察者并通知有效的观察者
        for (auto it = observers.begin(); it != observers.end();) {
            if (auto observer = it-&gt;lock()) {
                // 通知观察者
                &#43;&#43;it;
            } else {
                it = observers.erase(it);  // 移除失效的观察者
            }
        }
    }
    
    virtual ~Subject() {
        std::cout &lt;&lt; &#34;Subject destroyed\n&#34;;
    }
};
```

# 最佳实践

在设计类关系时, 就要明确所有权关系. 比如父子关系, 父对象使用share_ptr, 子对象使用weak_ptr指向父对象. 比如观察者模式中Subject持有Observer的weak_ptr, 同时定期检查weak_ptr是否过期, 清理无效的引用.

同时注意在使用weak_ptr会带来一定的性能开销, 因为要检查和上锁, 因此在多线程环境需要特别注意weak_ptr的使用, 最好的解决方案还是在设计时就避免循环引用. 


---

> 作者: [LGS](https://github.com/geekstormm)  
> URL: https://geekstormm.github.io/posts/shared_ptr/  

