---
tags:
  - 吉利
---


[~Bosch_Sync] 你好，测试是通过adb 命令修改的CC配置，是在工程模式设置中开启ADB模式后，然后用双公头线连接的电脑与车机端，通过adb命令

su  
busybox telnet 192.168.118.2  
root  
test_psis_car_cfg CFG_CAR_CONFIGURATION xx x 

来配置的CC


Commented by HU Yilang (XC-CT/ESA2-CN) on 2022-08-04T01:31:55.280Z:  
>testpsiscarcfg CFGCAR_CONFIGURATION xx x  
只是临时测试方法，方便开发快速验证功能  
该命令只修改了SOC侧的CC配置， 未修改SCC的cc配置  
  
因此当你使用正式方法--台架测试功能时，应当从台架输入CC数据到SCC侧，并记得退出SOC的测试模式：  
> psis_client -w -1 hut_cfg CFG_DEBUG_MODE 0  
> bosch_reset  
  
台架输入CC和手动命令行输入CC不能混用。  
UM=Abandoned再发送UM=Driving后， SCC会覆盖当前SOC的CC配置，这是正常的流程。