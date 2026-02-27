# Java Framework调试

**查看所有可调式的java进程**
```
adb jdwp
```

**查看对应进程的PID进程号**
```
adb shell ps -A | grep system_server
```

**建立端口转发**

```
adb forward tcp:8700 jdwp:1234
```