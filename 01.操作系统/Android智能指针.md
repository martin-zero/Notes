---
tags:
  - Android系统/Native
---

>Android的智能指针为了解决Android中Native C++指针手动管理的内存泄漏问题，与C++原生的智能指针类似，Android智能指针会自动管理对象的生命周期，退出作用域。Android中智能指针分为sp(强指针)与wp(若指针)两类，若想使用智能指针需要继承基类RefBase