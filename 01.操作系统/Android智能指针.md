---
tags:
  - Android系统/Native
---

>Android的智能指针为了解决Android中Native C++指针手动管理的内存泄漏问题，与C++原生的智能指针类似，Android智能指针使用**引用计数**的方式自动管理对象的生命周期，离开作用域自动释放内存。Android中智能指针分为`sp`(强指针)与`wp`(若指针)两类，使用智能指针需要继承`RefBase`类。

# 使用方法

```
```

# 原理