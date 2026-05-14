IPC通信
**Proxy端**客户端
Connect建立连接
RegNotify注册广播
StartNotify通知广播订阅

**Server端**服务端
Notify发送广播

**APPWatchdog** HAM 看门狗 崩溃拉起进程
**ClusterIVIServer** 与IVI交互
**GaugeServer** 表头信息 转速 功率。。。 阻尼处理
**HMIStateServer** 基于按键和HMI状态做处理
**Hotkey** mcu按键消息发送给对应模块
**Interface**  ？
**Logger** 封装MsgLog 日志系统 新增模块新增宏 START_关键日志
**PowerServer** 电源状态 自检状态
**Process** 实例化？
**SettingServer** 断电不丢失 psisReadKey表格
**SPICommServer服务**(SOC  <-> MCU FDBUS通信)低软SPI  switch中解析分发mcu中的信号
TCS