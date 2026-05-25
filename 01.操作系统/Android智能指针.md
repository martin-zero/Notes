---
tags:
  - Android系统/Native
---

> Android 的智能指针为了解决 Android 中 Native C++ 指针手动管理的内存泄漏问题，与 C++ 原生的智能指针类似，Android 智能指针使用**引用计数**的方式自动管理对象的生命周期，离开作用域自动释放内存。Android 中智能指针分为 `sp`（强指针）与 `wp`（弱指针）两类，想要被智能指针管理的类必须继承 `RefBase` 类。

# 使用方法

```c++
class Foo : public RefBase {
    public: void bar() {}
};

// 强引用
sp<Foo> sp1 = new Foo();
sp<Foo> sp2 = sp1; // 强引用计数 +1

// 弱引用（不增加强引用计数）
wp<Foo> wp1 = sp1;

// 弱引用提升为强引用（对象可能已销毁，需判空）
sp<Foo> promoted = wp1.promote();
if (promoted != nullptr) {
    promoted->bar();
}
```

## sp（强指针）

`sp<T>` 对应 C++ 的 `shared_ptr<T>`，持有对象的**强引用**，会增加对象的强引用计数（`mStrong`）。

- 构造时：强引用计数 +1，同时弱引用计数也 +1
- 析构时：强引用计数 -1；若强引用计数归零，对象被销毁；弱引用计数也随之 -1，若弱引用计数也归零，`RefBase` 控制块（`weakref_impl`）被释放
- 支持拷贝和赋值，语义与 `shared_ptr` 一致

## wp（弱指针）

`wp<T>` 对应 C++ 的 `weak_ptr<T>`，持有对象的**弱引用**，只增加弱引用计数（`mWeak`），不影响对象的生命周期。

- 构造时：弱引用计数 +1，**不增加**强引用计数
- 析构时：弱引用计数 -1
- 使用前必须通过 `promote()` 尝试提升为 `sp`，提升失败说明对象已被销毁
- 用于**打破循环引用**：若两个对象互相持有对方的 `sp`，则强引用计数永远不为零，造成内存泄漏；将其中一方改为 `wp` 即可解决

## RefBase

所有希望被智能指针管理的类都必须继承 `RefBase`。`RefBase` 内部维护一个 `weakref_impl` 控制块，存储强引用计数和弱引用计数：

```cpp
// 简化结构
class RefBase {
    weakref_impl* mRefs;  // 内部控制块，存放引用计数

    // 生命周期回调（可重写）
    virtual void onFirstRef();           // 第一次被 sp 持有时调用
    virtual void onLastStrongRef(...);   // 最后一个 sp 析构时调用
    virtual void onLastWeakRef(...);     // 最后一个 wp 析构时调用
};

struct weakref_impl {
    std::atomic<int32_t> mStrong;  // 强引用计数
    std::atomic<int32_t> mWeak;    // 弱引用计数
};
```

> **注意**：不要对同一个裸指针重复构造 `sp`，否则会导致引用计数错误，类似 `shared_ptr` 的双重释放问题。正确做法是从已有的 `sp` 拷贝，或在类内通过 `sp<T>(this)` 配合特定机制使用。

# 原理

# 与 C++ 智能指针对比

|维度|Android sp/wp|C++ shared_ptr/weak_ptr|
|---|---|---|
|所属标准|Android 私有（libutils）|C++11 标准库 `<memory>`|
|基类要求|必须继承 `RefBase`|无要求，任意类型均可|
|计数存储位置|存于对象内部（侵入式）|外部控制块（非侵入式）|
|强/弱引用计数|分离的 mStrong / mWeak|同一控制块两个字段|
|弱引用提升|`wp::promote()`|`weak_ptr::lock()`|
|独占所有权|无（无 unique_ptr 对应）|有 `unique_ptr`|
|生命周期回调|onFirstRef / onLastStrongRef|无内置回调|
|线程安全|计数原子操作，线程安全|计数原子操作，线程安全|
|自定义删除器|不支持|支持（模板参数）|
|make_shared 等价|无（new 对象后直接赋给 sp）|`std::make_shared<T>()`|
|数组支持|不支持|C++17 起支持|
|跨平台可用性|仅 Android / AOSP|全平台|

> 核心差异：Android sp/wp 是**侵入式**引用计数——计数嵌入对象本身（通过继承 `RefBase`）；C++ `shared_ptr` 是**非侵入式**——计数存于独立控制块，对象本身无感知。

## 何时选用 Android sp/wp

- 在 **AOSP / Android Native** 层开发（Framework、HAL、JNI 等），优先使用 `sp/wp`，与系统代码风格一致，且无需额外引入标准库
- 纯 C++ 项目或跨平台代码，优先使用标准库的 `shared_ptr/weak_ptr`，兼容性更好，且支持更多特性（自定义删除器、`make_shared`、数组等）