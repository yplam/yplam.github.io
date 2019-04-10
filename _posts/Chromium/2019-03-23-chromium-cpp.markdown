---
title:  "Chromium 源码阅读：C++ 基础"
categories: Chromium, C++
---

阅读 Chromium 源码的过程中，发现最大的障碍在于自己对 C++ 的理解还停留在大学时期，平时使用 C++ 也
仅仅停留在写个简单类，单例、工厂等简单的模式；所以在此针对 Chromium 源码中自己未曾用过的编程方式
做一个笔记。

## std:move

std:move 可以将 lvalue 转换成 rvalue，减少对象复制，提高效率。

lvalue: 有名称的、在内存中有实际位置供程序员存取的物件。

rvalue：没有名称的临时物件，离开当前语句后即被摧毁，如 f(a+b) 中的 a+b

C++ 11 提供针对 rvalue 引用的方法重载，因为拿到的是 rvalue，那么在方法内部就可以随便把 rvalue
的内容拿来用，因为我们用完后 rvalue 也没有其他用途了，所以可以跳过复制的步骤，提高效率。


```
class string
{
    char* data;

public:

    string(const char* p)
    {
        size_t size = strlen(p) + 1;
        data = new char[size];
        memcpy(data, p, size);
    }
    
    ~string()
    {
        delete[] data;
    }

    string(const string& that)
    {
        size_t size = strlen(that.data) + 1;
        data = new char[size];
        memcpy(data, that.data, size);
    }
    
    string(string&& that)   // string&& is an rvalue reference to a string
    {
        data = that.data;
        that.data = nullptr;
    }
}

```

参考：
* [[C++] rvalue reference 初入門](https://shininglionking.blogspot.com/2013/06/c-rvalue-reference.html)
* [What are move semantics?](https://stackoverflow.com/questions/3106110/what-are-move-semantics)

