## IPC通信 模块之间也用IPC通信
**Proxy端**客户端
Connect建立连接
RegNotify注册广播
StartNotify通知广播订阅

**Server端**服务端
Notify发送广播

## 模块
**Middleware**
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
	**TCServer** 行车数据相关 里程、保养信息。。
	**TelltaleServer** TT灯
	**Timer** 定时器
	**Utils** 公式转换 单位转换
	**VehicleComm** 与IVI交互
	**WarningServer** 从SPI模块转发给HMI Lib
	**Script** 进程启动脚本

**sdk**
	**external** proto文件 无用了
	**testApp** 测试用 无用

**HMI** 
	**hmi** kanzi文件
	**kanzi** kanzi工程
	**Lib** kanzi库
