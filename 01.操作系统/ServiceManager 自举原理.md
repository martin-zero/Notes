---
tags:
  - Android系统
  - ServiceManager
原文链接: https://mp.weixin.qq.com/s/fbaovmnp4Im7uwej9pnYSQ
---
## 核心一句话

> ServiceManager 不是被注册的，它是被 Binder 驱动通过 `BINDER_SET_CONTEXT_MGR_EXT` 这条 ioctl **钦点**的。固定句柄 0，全局唯一，系统启动比 zygote 还早。

---

## 第一层：谁拉起 ServiceManager？

- **拉起者**：init 进程（pid 1）解析 `servicemanager.rc`
- **关键配置项**：
    - `class core`：早期服务组，zygote 启动前就跑起来
    - `critical`：挂了 init 重启它；反复挂触发整机重启
    - `onrestart restart audioserver/gatekeeperd/hal...`：SM 重启时整组依赖服务跟着重启，避免缓存的 binder 引用变成野指针
- **加分答法**：SM 是 init 直接 fork 的 native 进程，不走 zygote，没有预加载 Java 类

---

## 第二层：main() 干了什么？

|步骤|代码|要点|
|---|---|---|
|① 打开驱动|`ProcessState::initWithDriver("/dev/binder")`|mmap 约 1MB，这是 Binder 单次事务上限的物理来源|
|② 禁线程池|`setThreadPoolMaxThreadCount(0)`|绝对串行，单线程处理所有服务发现，够用且避免锁|
|③ 自注册|`manager->addService("manager", manager)`|为了 `dumpsys -l` 列表对称，非自举核心|
|**④ 加冕**|`ps->becomeContextManager()`|**自举核心**：ioctl 告诉驱动"handle=0 的事务都路由给我"|
|⑤ 事件循环|`Looper::pollAll(-1)`|永久阻塞，等事务|

**加冕的本质**：驱动内维护全局变量 `binder_context_mgr_node`，整个生命周期**只能设置一次**，内核保证排他，且要求 uid = AID_SYSTEM。

---

## 第三层：客户端怎么找到 SM？

```
defaultServiceManager()
  → ProcessState::getContextObject(nullptr)
    → new BpBinder(0)
      → Binder 驱动：handle=0 → 路由到 binder_context_mgr_node
```

- `while + sleep(1)` 的 retry 循环是工程稳健性：SM 可能还没就绪
- 所有进程第一次拿 SM，都是构造 `BpBinder(0)`，没有额外发现步骤

---

## 「鸡生蛋」的解法

上层没法解的自举问题，**下沉到内核硬编码**。SM 不走"先找 SM 再注册"的普通流程，而是让 Binder 驱动给它开后门，这是**约定优于配置的内核级实现**。

---

## SM 崩溃会怎样？

`critical` 标记让 init 重启 SM，同时 `.rc` 里的 `onrestart restart` 把所有缓存了旧 binder 引用的 native 服务一并重启。Java 层 SystemServer **不会**自动重启，线上稳定恢复通常需要整机软重启。