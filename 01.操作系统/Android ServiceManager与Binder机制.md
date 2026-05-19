---
tags:
  - Android系统
原文链接: https://mp.weixin.qq.com/s/fbaovmnp4Im7uwej9pnYSQ
---
上周三晚上九点半，深圳南山一家 ROM 厂商现场面，面试官姓张。

聊了大半天 Binder。从一次拷贝聊到 mmap，从 AIDL 的 Stub/Proxy 聊到 oneway 调用，张工点头点得挺勤快，我心里有点飘。

然后他放下笔，慢悠悠问了一句：

> 「那你说说看，ServiceManager 自己也是个 Binder server。它启动那一刻，Binder 驱动里**还没人注册过任何服务**，它怎么把自己注册成那个谁都能找到的总管？」

我卡住了。

倒不是不知道 ServiceManager 是 Binder 体系的根。是那一瞬间我突然意识到——这丫的是个**鸡生蛋、蛋生鸡**的问题。

ServiceManager 的职责是给所有 Binder 服务发"114 黄页"。可它自己也得是一个 Binder 服务，才能被别的进程跨进程调到。**那它启动的时候，谁来给它发那张黄页？**
回家之后我把这道题翻了个底朝天。看xx的视频复习了一遍，又顺手翻 AOSP 的 `frameworks/native/cmds/servicemanager/` 目录，把里面拢共 200 多行的 `main.cpp` 一行一行抠了。

抠完笑出了声。

抠完才发现，这段代码漂亮到不像是 Google 写的——是漂亮到**像哪个老师傅花了一周时间专门琢磨出来的**那种漂亮。**这道"自举"题，AOSP 早就解决了，而且解得相当朴素**。

今天这篇就是把那一晚的复盘原汁原味发出来。我会带你走完三层：

- 第一层：ServiceManager 是被谁拉起来的？（init.rc 视角）
    
- 第二层：ServiceManager 的 main 函数到底干了什么？（C++ 源码视角）
    
- 第三层：客户端怎么找到 ServiceManager？（"句柄 0"的魔法）
    

读完你下次再被这种自举题问到，敢笑出声。

---

## 第一层考点：ServiceManager 是被谁、在哪个时间点拉起来的？

我让团队的小伙子答这题，他答："系统启动的时候吧。"

我说：兄弟，这答案约等于没答。**系统启动有一百个阶段**，你得说清楚是哪一阶段。

ServiceManager 是 Android 启动链路里的一个关键节点。整个链路从下往上是：

```
bootloader  
  ↓  
linux kernel 启动  
  ↓  
init 进程 (pid 1，第一个用户态进程)  
  ↓ 解析 init.rc  
  ↓  
servicemanager 进程  <-- 我们在这里  
  ↓  
zygote 进程  
  ↓  
SystemServer 进程  (AMS / WMS / PMS 都在这里)  
  ↓  
Launcher 进程
```

**关键就在 init 解析 init.rc 那一步**。Android 的 init 进程会读 `/system/etc/init/` 目录下一堆 .rc 文件，其中有一个 `servicemanager.rc`，长这样（Android 14 真实路径 `frameworks/native/cmds/servicemanager/servicemanager.rc`）：

```
service servicemanager /system/bin/servicemanager  
    class core animation  
    user system  
    group system readproc  
    critical  
    file /dev/kmsg w  
    onrestart setprop servicemanager.ready false  
    onrestart restart --only-if-running apexd  
    onrestart restart audioserver  
    onrestart restart gatekeeperd  
    onrestart class_restart --only-enabled main  
    onrestart class_restart --only-enabled hal  
    onrestart class_restart --only-enabled early_hal  
    writepid /dev/cpuset/system-background/tasks  
    shutdown critical
```

我把这段配置里最关键的几条圈给你：

**`critical`** ——这个进程被标记成"关键服务"。意思是它要是挂了，init 会**重启它，并且如果短时间内反复挂，整个手机会触发重启**。Android 的设计哲学很简单：没了 ServiceManager，整套 Binder 就废了，那系统也没必要继续跑了。

**`onrestart restart audioserver / gatekeeperd / hal ...`** ——一旦 ServiceManager 重启，这一长串的 native 服务（audio、gatekeeper、HAL 全家桶）也跟着重启。为啥？因为这些服务里都缓存了**ServiceManager 的 binder 引用**。ServiceManager 一旦重启，那些缓存就成了野指针。所以宁可大家一起重来，也不留隐患。

这条配置是 framework 的"群体免疫"——很经典的工程思路。

**`class core animation`** ——init.rc 里一堆服务都标了 class，class 的作用是按组启动。`core` 这个 class 在系统启动早期就被触发，所以 ServiceManager 是早期服务里第一批被拉起来的。具体早到什么程度？早到 **zygote 还没启动**。

这就回答了第一层：ServiceManager 是 init 进程在系统启动早期，按 .rc 配置直接拉起来的，比 zygote 还早。

> **面试加分点**：你可以补一句——"servicemanager 跟 zygote、surfaceflinger、media.audio_flinger 这些都是 init 直接拉起来的 native 服务，**不走 zygote fork**。所以它们不共享 zygote 预加载的那些 Java 类——它们是纯 native 进程。"

讲到这儿张工大概已经会点头了。但真正的硬骨头在下一层。

---

## 第二层考点：ServiceManager 的 main 函数到底干了什么？

> AOSP 源码路径：`frameworks/native/cmds/servicemanager/main.cpp`（Android 14，约 90 行）

注意一个事实：这玩意儿在 Android 9 之前是用 C 写的（`service_manager.c`，更难读），9 之后改成了 C++（`main.cpp` + `ServiceManager.cpp`）。咱们看 14。

把这 90 行的核心摘出来：

```cpp
int main(int argc, char** argv) {  
    if (argc > 2) {  
        LOG(FATAL) << "usage: " << argv[0] << " [binder driver]";  
    }  
  
    const char* driver = argc == 2 ? argv[1] : "/dev/binder";  
  
    sp<ProcessState> ps = ProcessState::initWithDriver(driver);  
    ps->setThreadPoolMaxThreadCount(0);  
    ps->setCallRestriction(ProcessState::CallRestriction::FATAL_IF_NOT_ONEWAY);  
  
    sp<ServiceManager> manager = sp<ServiceManager>::make(std::make_unique<Access>());  
    if (!manager->addService("manager", manager, false /*allowIsolated*/,  
                              IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT).isOk()) {  
        LOG(ERROR) << "Could not self register servicemanager";  
    }  
  
    IPCThreadState::self()->setTheContextObject(manager);  
    ps->becomeContextManager();  
  
    sp<Looper> looper = Looper::prepare(false /*allowNonCallbacks*/);  
  
    BinderCallback::setupTo(looper);  
    ClientCallbackCallback::setupTo(looper, manager);  
  
    while(true) {  
        looper->pollAll(-1);  
    }  
  
    return EXIT_FAILURE;  
}
```

短短二十几行。咱们一段一段拆。

### 拆第 1 段：打开 /dev/binder

```cpp
sp<ProcessState> ps = ProcessState::initWithDriver(driver);
```

`ProcessState` 是 native binder 里管"本进程的 Binder 状态"的类。它是**进程级单例**，每个用了 Binder 的进程开第一次的时候，都会构造一个。

构造里干的事儿就一句话：**打开 `/dev/binder` 设备节点，然后把它 mmap 到本进程的地址空间**。这块 mmap 区域默认大小是 `(1*1024*1024) - sysconf(_SC_PAGE_SIZE) * 2`——大约就是 1MB 减两个页。这就是面试时被问烂的"Binder 一次事务上限 1MB"的物理来源。

**记忆点**：Binder 一次事务上限 1MB 不是协议规定的，是 ProcessState 构造时 mmap 区域的大小（1MB 减两个页）。这是物理限制，不是软规则。

但是 ServiceManager 这里有个**特殊配置**：

```cpp
ps->setThreadPoolMaxThreadCount(0);
```

普通进程的 binder 线程池默认会有 15 个线程并发处理事务。**ServiceManager 把这个数字设成 0**——意思是：我不开额外的线程池，整个进程**单线程跑**，所有事务排着队来。

这是 ServiceManager 的设计哲学：**绝对的串行、绝对的轻量、绝对不并发**。它要保证全局唯一一致性，并发反而会引入锁开销和正确性风险。一个手机里所有进程的服务发现请求都要走它，但因为每个请求处理时间极短（无非就是查个 map），单线程足够扛。

> 这个细节挺多人不知道的。下次被问"为啥 ServiceManager 不开线程池"，你可以慢悠悠把这个原因说出来。

### 拆第 2 段：自己把自己加进 map（神操作）

```cpp
sp<ServiceManager> manager = sp<ServiceManager>::make(std::make_unique<Access>());
manager->addService("manager", manager, ...);
```

注意第二行——**ServiceManager 在自己的服务表里，也注册了一份自己**，名字叫 "manager"。

很多人看到这一行会愣住：你都已经是 ServiceManager 了，为啥还要往自己里面塞一个自己？

这是个**为了 dumpsys 命令工作得统一**而搞的工程小动作。当你在 adb shell 里敲 `dumpsys -l`，它列出所有已注册服务的时候，ServiceManager 自己也得在那张表里出现，否则 dumpsys 会少一行。这种"自己也是服务"的对称性看着多余，实际上让上层工具的代码逻辑统一了——所有服务一律走查表，没有特殊情况。

这一行不是**自举的关键步骤**。真正的自举在下一行。

### 拆第 3 段：真正的自举——becomeContextManager()

```cpp
IPCThreadState::self()->setTheContextObject(manager);
ps->becomeContextManager();
```

这才是回答张工那道题的灵魂代码。

`becomeContextManager()` 干了一件事：**它通过 ioctl 系统调用，告诉 Binder 驱动——"嘿，老子是这台机器的 ContextManager（上下文管理器），从现在开始，所有发送给 handle = 0 的事务，都路由给我"**。

代码大致是这样的（在 `ProcessState.cpp` 里）：

```cpp
bool ProcessState::becomeContextManager() {  
    // ...  
    flat_binder_object obj { .flags = FLAT_BINDER_FLAG_TXN_SECURITY_CTX };  
    int result = ioctl(mDriverFD, BINDER_SET_CONTEXT_MGR_EXT, &obj);  
    // ...  
}
```

`BINDER_SET_CONTEXT_MGR_EXT` 这条命令进了 Binder 驱动以后，驱动内部维护了一个全局变量 `binder_context_mgr_node`——这是个指向 ServiceManager 那个 binder 实体的指针。**这个指针在 Binder 驱动整个生命周期里只能被设置一次**，由内核保证排他。

设置完之后，**handle = 0 这个特殊编号在 Binder 驱动里被永久绑定到了 ServiceManager**。

——这就是答案。

ServiceManager 不是被某个"更高一级的服务"注册的，**它是被 Binder 驱动以一个特殊系统调用单独"加冕"的**。这个加冕动作是 Binder 驱动给 ServiceManager 开的后门，整个系统只有这一个进程能走这条路（驱动会做权限校验，要求 uid = AID_SYSTEM）。

**记忆点**：「鸡和蛋」问题的解法——让一只鸡有特权，钦点它当祖宗。ServiceManager 不是被注册的，是被 Binder 驱动用 ioctl 直接加冕的。

我跟你讲，第一次理解到这里的时候，我手机都差点摔了——这设计是真的精妙。所有其他服务的注册都得走"先找到 ServiceManager 再调用 addService"的流程，唯独 ServiceManager 自己，是被驱动直接钦点的。

> **面试加分点**：「这个设计的好处是 Binder 驱动只需要硬编码一个 handle = 0 的特殊路由，就把整个服务发现的入口固化下来了。所有 native 进程、所有 Java 进程，第一次创建 BpBinder 的时候都用 0，都能瞬间找到 ServiceManager。这是一种**约定优于配置**的内核级实现。」

---

## 第三层考点：客户端是怎么找到 ServiceManager 的？

服务端钦点完了，客户端这边其实就简单了。我贴一段你天天用但可能没细看的代码——**`getService` 内部是怎么走的**：

```cpp
// frameworks/native/libs/binder/IServiceManager.cpp  
sp<IServiceManager> defaultServiceManager() {  
    if (gDefaultServiceManager != nullptr) return gDefaultServiceManager;  
  
    // ...  
    while (gDefaultServiceManager == nullptr) {  
        gDefaultServiceManager = interface_cast<IServiceManager>(  
            ProcessState::self()->getContextObject(nullptr));  
        if (gDefaultServiceManager == nullptr) sleep(1);  
    }  
  
    return gDefaultServiceManager;  
}
```

注意这一行：`ProcessState::self()->getContextObject(nullptr)`。

这个 `getContextObject` 内部最终会构造一个 **`new BpBinder(0)`**——也就是一个**指向 handle 等于 0 的远端 binder 引用**。

这个 BpBinder 看着是个普通的 C++ 对象，但每次你调它的 `transact()`，最后都会走到 Binder 驱动里。驱动一看 handle 是 0，就直接把事务路由到那个被钦点过的 `binder_context_mgr_node`，也就是 ServiceManager 进程。

注意上面那个 `while + sleep(1)` 的 retry 循环——这是个**真实工程细节**。客户端进程启动的时候，ServiceManager **可能还没准备好**（虽然 init.rc 里 ServiceManager 启动得很早，但仍有极小概率被并发 race 到）。这时候 `getContextObject` 会返回 nullptr，于是 sleep 1 秒再试，直到拿到为止。

这种"系统级 retry"在 framework 里到处都是——一开始我也觉得啰嗦，写多了才知道是必备的稳健性设计。

> 顺手补一刀的奇案：**ServiceManager 死了会发生什么？**
> 
> 我之前给团队做培训的时候出过这道题——大家都是猜的。
> 
> 答案：那台手机基本就废了。`servicemanager.rc` 里的 `critical` 标记会让 init 把它重启起来，但是**所有依赖 ServiceManager 的 native 服务（audio / hal / gatekeeper 那一堆）都缓存了过期的 binder 引用**——这就是为什么 .rc 文件里写了一长串 `onrestart restart`，把整组依赖服务一并拉起来重新注册。
> 
> 即便如此，**Java 层的 SystemServer 不会被自动重启**。所以最稳的恢复路径是整机软重启。线上你要是真碰到 servicemanager 反复挂的现象，恭喜你——99.9% 是底层驱动或者 selinux 策略出问题，得让 BSP 团队顶上。

---

## 把这一晚的复盘锁在一句话里

> ServiceManager 不是被注册的，**它是被 Binder 驱动通过 `BINDER_SET_CONTEXT_MGR` 这条 ioctl 钦点的**。它跑在 init 直接拉起来的 native 进程里，单线程串行处理所有服务发现请求，固定句柄 0。所有其他进程拿 ServiceManager 的方式，都是构造一个 `BpBinder(0)`，然后由 Binder 驱动把事务路由过去。

把这句话背下来，下次再被问"自举"题，你笑一下，慢慢说。

---

## 团队点评

> **老韩（AOSP 架构视角）**：「Google 这套设计抓住了一个本质——服务发现机制必须是**自洽的**。如果 ServiceManager 也走"先找 ServiceManager 再注册"的流程，就是无限递归。所以他们把这个根问题甩给了 Binder 驱动，让内核来背这个'第一只鸡'的锅。这是典型的'层次跨越'解法——上层没法解决的问题，下沉到下层去硬编码。」

> **Kevin（面试官视角）**：「这道题在大厂 framework 岗的二面、三面是必问的。如果候选人能从 init.rc 一路讲到 ioctl，能扯清楚 BINDER_SET_CONTEXT_MGR 的角色，基本上 framework 的基础就稳了——能答这个的，下一道就敢往 mmap 的内存管理深挖。」

---

## 面试一答到底

> **Q1**：ServiceManager 是怎么启动的？
> 
> **A**：它是 init 进程通过解析 `frameworks/native/cmds/servicemanager/servicemanager.rc` 直接拉起来的 native 进程，比 zygote 还早，标记为 `critical`，挂了 init 会自动重启。

> **Q2**：ServiceManager 自己也是 Binder server，谁来注册它？
> 
> **A**：它没被任何更高一级的"注册中心"注册过——它通过 `BINDER_SET_CONTEXT_MGR_EXT` 这条 ioctl 被 Binder 驱动**直接钦点为 ContextManager**。整个 Binder 驱动里只能有一个 ContextManager，对应的 handle 永久固定为 0。这是 Android Binder 体系自举（bootstrap）的根。

> **Q3**：客户端在自己进程里第一次调 `IServiceManager`，是怎么找到 ServiceManager 进程的？
> 
> **A**：通过 `defaultServiceManager()` → `ProcessState::getContextObject(nullptr)` → 构造 `new BpBinder(0)`。Binder 驱动一看 handle 是 0，就把事务路由到 ContextManager（也就是 ServiceManager 进程）。这条路径是 Binder 驱动写死的特殊路由。

> **Q4**：为什么 ServiceManager 不开 binder 线程池？
> 
> **A**：源码里调用了 `setThreadPoolMaxThreadCount(0)`。设计哲学是绝对串行——所有服务注册和查询请求排队处理，每个请求耗时极短（一次 map 查询），单线程不仅够用，还能避免锁开销和并发正确性风险。这是性能与简单性之间最舒服的折中。> **Q5**：如果 ServiceManager 进程崩了，会发生什么？
> 
> **A**：servicemanager.rc 里标了 `critical`，init 会自动重启它。但因为其他 native 服务（audio、gatekeeper、HAL 全家桶）都缓存了 ServiceManager 的 binder 引用，重启后这些缓存全失效。所以 .rc 配置里写了一堆 `onrestart restart` 把这组依赖服务一并拉起来重新注册。Java 层的 SystemServer 不会自动跟着重启，线上稳定恢复一般要触发整机软重启。

---

## 留个钩子

那次面试结束，张工冷笑了一下：

> 「不错，这套你能答到 ioctl 已经超过我们今年面过的 80% 候选人了。但你答得太顺了，听着像背的——你能不能给我**手撕一段** ServiceManager 的 main 函数，从打开 /dev/binder 到 `becomeContextManager`，把里面调用关系画在白板上？」

我那天没画出来。回家翻 ProcessState 翻到凌晨。

明天那篇我把这个锅补完——**`/dev/binder` 这个设备节点，到底是怎么被打开、被 mmap、被驱动认证身份的**。讲完你以后看到 binder 内核驱动代码就不再发怵了。

下篇见。