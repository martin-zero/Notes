# Java Framework调试
#### 建立连接

**查看所有可调式的java进程**
```sh
adb jdwp
```

**查看对应进程的PID进程号**
查看system_server的进程号
```sh
adb shell ps -A | grep system_server
```

**建立端口转发**
将电脑8700端口与Android进程号为1234的进程相连接
```sh
adb forward tcp:8700 jdwp:1234
```

**启动jdb并attach**
```sh
jdb -attach localhost:8700
```

加载源码路径