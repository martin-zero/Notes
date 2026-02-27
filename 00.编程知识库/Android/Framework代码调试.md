
# Java Framework调试

## 建立连接

#### 查看所有可调式的java进程
```sh
adb jdwp
```

#### 查看对应进程的PID进程号
查看system_server的进程号
```sh
adb shell ps -A | grep system_server
```

#### 建立端口转发
将电脑8700端口与Android进程号为1234的进程相连接
```sh
adb forward tcp:8700 jdwp:1234
```

#### 启动jdb并attach
通过8700端口进行jdb调试
```sh
jdb -attach localhost:8700
```

#### 加载源码路径
进行断点调试需要加载源码路径
- 可以使用`use`查看当前源码路径
- 可以添加多个源码路径
```jdb
use frameworks/base/services/core/java
```

## 代码调试

#### 通过方法名设置断点
```jdb
stop in com.android.server.am.ActivityManagerService.systemReady
```

#### 通过行号设置断点
```jdb
stop at com.android.server.am.ActivityManagerService:7421
```

#### 查看断点
```jdb
clear
```

#### 继续执行
```jdb
cont
```

#### 查看调用栈
```jdb
where
```

#### 查看局部变量
```jdb
locals
```

#### 查看对象
```jdb
print obj
```

#### 单步执行
```jdb
step     // 进入函数
next     // 不进入函数
```

#### 查看线程
```
threads
thread <id>
```

# Native Framework调试

## 建立连接

#### 启动调试代理
在目标设备上启动 LLDB 远程调试服务，并附加到542进程，暴露到5039端口
```sh
adb shell
su
lldb-server g :5039 --attach 542
```

#### 建立端口转发
将电脑的5039端口与设备的5039端口建立连接
```sh
adb forward tcp:5039 tcp:5039
```

## 代码调试


# 常见问题

