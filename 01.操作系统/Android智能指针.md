---
tags:
  - Android系统/Native
---

>Android的智能指针为了解决Android中Native C++指针手动管理的内存泄漏问题，与C++原生的智能指针类似，Android智能指针使用**引用计数**的方式自动管理对象的生命周期，离开作用域自动释放内存。Android中智能指针分为`sp`(强指针)与`wp`(若指针)两类，想要被智能指针管理的类必须继承`RefBase`类。

# 使用方法


```c++
class Foo : public RefBase {
	public: void bar() {} 
};

// 强引用
sp<Foo> sp1 = new Foo();
sp<Foo> sp2 = sp1; // +1

// 弱引用 + 提升
wp<Foo> wp1 = sp1;
sp<Foo> promoted = wp1.promote();
if (promoted != nullptr) {
    promoted->bar();
}

```

# 原理

# 与C++智能指针对比

|维度|Android sp/wp|C++ shared_ptr/weak_ptr|
|---|---|---|
|所属标准|Android 私有（libutils）|C++11 标准库 <memory>|
|基类要求|必须继承 `RefBase`|无要求，任意类型均可|
|计数存储位置|存于对象内部（侵入式）|外部控制块（非侵入式）|
|强/弱引用计数|分离的 mStrong / mWeak|同一控制块两个字段|
|弱引用提升|`wp::promote()`|`weak_ptr::lock()`|
|独占所有权|无（无 unique_ptr 对应）|有 `unique_ptr`|
|生命周期回调|onFirstRef / onLastStrongRef|无内置回调|
|线程安全|计数原子操作，线程安全|计数原子操作，线程安全|
|自定义删除器|不支持|支持（模板参数）|
|make_shared 等价|无（new 对象后直接赋给 sp）|std::make_shared<T>()|
|数组支持|不支持|C++17 起支持|
|跨平台可用性|仅 Android / AOSP|全平台|

> 核心差异：Android sp/wp 是**侵入式**引用计数——计数嵌入对象本身（通过继承 RefBase）；C++ shared_ptr 是**非侵入式**——计数存于独立控制块，对象本身无感知。