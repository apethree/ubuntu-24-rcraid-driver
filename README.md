# 笔记本 linux fakeraid驱动安装教程

[English version CHECK ME! :)](README.en.md)

## 引入

这个驱动安装器只支持`Ubuntu 22.04.1`捏，
官方写的脚本很糟糕，不符合国情，处于完全不能安装的地步，
包括但不限于无法访问谷歌导致超时、apt软件名错误导致不能安装等

按理说只要是`5.15.0`内核应该都能跑，
哥们还是`linux`小萌新，如果有大佬能解决其他发行版的话欢迎pr，阿里嘎多

为了避免嫌我啰嗦，我就把日记放在后面了，[一件直达](#_4)（bushi

## 安装

1. 下载

## 排错

如果报错说明是


## 日记

某天没事做翻`bios`发现自己笔记本除了`AHCI`的模式，还支持`RAID`模式嘞，
奔着想要搞个`raid0`来提提速的想法，索性就来试一试这个传说中的笔记本`raid`模式
[^1][^3]

<!-- ![](https://kmpic.asus.com/images/2021/09/30/f1c03f6d-af06-4366-bb94-685e98528901.png) -->
![asus-bios-sata-mode.png](pic/asus-bios-sata-mode.png)

根据`arch`教派的亲传[^2]，
这玩意可以通过`dmraid`来安装驱动来识别`raid`组，
不过这个貌似只支持牢英的`Intel Rapid Storage Technology`，
而且也被废弃了，目前牢英官方推荐使用`mdadm`进行软raid抽象层管理，
`dmraid`是通过`/dev/mapper`来管理设备的，而不是`/dev/sdx`，
下面不再使用`dmraid`安装，采用`AMD`官方的`rcraid`驱动管理。

### bios 选择

一般`DIY`主板用户在对应平台下载`raid驱动包`即可，
我是`天选4R`(AMD YES)，故就在对应官网笔记本驱动下载。
详细过程非常简单，官方有一系列的操作图示，跟着就行。

> 1. 进入 BIOS 设定: 计算机重新启动时,在 POST\(开机自动测试\) 时按下 `F2`，进入BIOS 设定页面。
> 2. 当进入 BIOS 设定画面时, 将会出现计算机系统信息。之后在页面中按下 `F7` 进入 \[Advanced Mode\] \(进阶模式\)。
> 3. 在 \[Advanced Mode\] 设定页面中，选择 \[SATA Configuration\] 按下 `Enter` 进入。
> 4. 在 \[SATA Configuration\] 页面中，将 \[SATA Mode Selection\] 项目切换为 \[RAID\]，之后按下 `F10` 储存并离开BIOS设定，即完成开启 \[RAIDXpert2 Configuration Utility\]。
> 5. 重复1-2步骤进入BIOS 设定页面，选择 \[RAIDXpert2 Configuration Utility\] 按下 `Enter` 进入。
> 6. 在 \[RAIDXpert2 Configuration Utility\] 页面中，选择 \[Array Management\] 然后按下 `Enter`。
> 7. 在 \[Array Management\] 页面中，选择 \[Delete Array\] 项目后按下 `Enter`。
> 8. 在 \[Delete Array\] 页面中，将选定的RAID 数组切换至 \[On\] 或全选为 \[Check All\]，之后选择 \[Delete Array\(s\)\] 项目按下 `Enter`。
> 9. 最后在 \[Warning\] 页面中再次确认，完成后切换 \[Confirm\] 项目至 \[On\]，之后选择 \[Yes\] 开始执行删除。

<!-- ![](https://kmpic.asus.com/images/2021/09/30/cba55582-2e90-4612-b653-f2b39726e3e6.png) -->
![asus-bios-array.png](pic/asus-bios-array.png)

### 双系统 安装(Win侧)

#### 方案一：直接使用 Windows 安装镜像

![win-install]


[^1]: 幻16逆天跑分：https://www.bilibili.com/video/BV17W4y1h7mT
[^2]: fakeraid介绍：https://wiki.archlinuxcn.org/wiki/RAID#实现方式
[^3]: 华硕笔记本raid教程：https://www.asus.com.cn/support/faq/1046579/#cc
[^4]: 官方Windows rcraid驱动下载：https://dlcdnet.asus.com/pub/ASUS/GamingNB/DriverforWin10/Raid_ROG/Raid_windows_driver_930_00266.zip


C:.
├─pic
├─ubuntu
│  ├─KERNEL6+: manully patch
│  ├─manager: https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/RAIDXpert2_Linux_932_00230.zip
│  └─UBUNTU22.04.1: https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/RAID_Linux_Ubuntu_2.2.40-9.3.2.00230.zip
└─windows
    ├─manager: https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/RAIDXpert2_Windows_932_00195.zip
    ├─ROG: https://dlcdnet.asus.com/pub/ASUS/GamingNB/DriverforWin10/Raid_ROG/Raid_windows_driver_930_00266.zip
    ├─WIN10: https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/Raid_PKG_S0i3_win_9.3.2_00158.zip
    └─WIN11: https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/RAID_Windows_Driver_932_00187.zip