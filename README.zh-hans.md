笔记本 fakeraid 驱动安装教程
=================================

引入
---------

官方写的脚本很糟糕，处于完全不能安装的地步，
包括但不限于无法访问谷歌导致超时、apt软件名错误导致不能安装等。

目前内核 5.x - 6.14 都能跑

为了避免嫌我啰嗦，我就把详细教程放在后面了，[一件直达](#手把手超详细双系统安装教程)（bushi


安装二进制驱动
---------

- 官方驱动包

先决条件: Ubuntu 22.04.1 

下载安装[官方驱动包](https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/RAID_Linux_Ubuntu_2.2.40-9.3.2.00230.zip)
或[备用下载](linux/UBUNTU22.04.1/x64.tar.zst)，根据里面附带的说明书执行即可。

> 预安装阶段:
> 启动安装镜像,在`Grub2`界面`Try and Install Ubuntu`选项上按`e`键进入编辑模式
> 在`/casper/vmlinuz`的`splash quiet ---`处修改为`modprobe.blacklist=ahci,nvme break=mount ---`
> 按下`F10`开始引导，正常情况进入`initramfs`阶段
> ```bash
> mkdir -p /tmp/dd
> mount -t vfat /dev/sxx /tmp/dd	# sxx是你驱动包路径(挂载分区格式以自己为准)
> /tmp/dd/preinstall
> ```
> 
> 安装 Ubuntu Installer(不要立即重启)
>
> 安装结束工作阶段:
> ```bash
> sudo su -	# (sudoers应该允许su) 或 sudo -i(sudoers应该允许bash)
> mkdir -p /tmp/ddp
> mount -t vfat /tmp/ddp
> /tmp/ddp/postinstall
> ```
> 重启完成！


- 新内核补丁驱动包

先决条件: 6.14内核版本的所有`Linux`发行版(如果你有空帮我把每个版本都编译一下)\
TODO: 拿 github action workflow 跑些通用的长期维护内核用

下载安装[驱动包](linux/KERNEL6+/x64.tar.zst)，
正常方式用`rufus`、`Balena Etcher`或`dd`烧录系统镜像到可移动介质，
烧录完成后在介质内路径`/boot/grub/`里修改`grub.cfg`和`loopback.cfg`，
其在每个`cfg`的`linux	/casper/vmlinuz`后追加`modprobe.blacklist=ahci,nvme`。

启动安装镜像，正常进入安装模式，在使用安装器分区之前，
打开`Terminal`终端挂载驱动包，运行里面自带的ko。
```bash
mkdir -p /tmp/dd
# 挂载装有驱动的文件夹(分区格式视具体情况而定)
sudo mount -t vfat /dev/sxx /tmp/dd
sudo insmod /tmp/dd/rcraid.ko
```

按理说此时注入成功将没有报错，即未发生`Kernel Panic`，
详细自行查看`sudo dmesg | tail -20` 或 `modinfo rcraid`。

正常安装系统到阵列中。

安装完毕之后打开`terminal`将ko驱动打入安装好的系统中：
```bash
# 如果之前没有卸载驱动文件夹的话，否则执行2、3条
sudo cp /tmp/dd/rcraid.ko /target/lib/modules/`uname -r`/kernel/drivers/scsi/rcraid.ko
# 挂载装有驱动的文件夹(分区格式视具体情况而定)
sudo mount -t vfat /dev/sxx /tmp/ddp
# 拷贝到内核模块里
sudo cp /tmp/ddp/rcraid.ko /target/lib/modules/`uname -r`/kernel/drivers/scsi/rcraid.ko
# 切换到目标系统的根环境
sudo chroot /target
```
进入系统`chroot`视图:
```bash
# 生成模块的依赖关系和映射文件
depmod -a `uname -r`
# 备份启动镜像 勇的话可以不用
cp -ap /boot/initrd.img-`uname -r` /boot/initrd.img-`uname -r`.bak
# 构建新的启动ramdisk内存文件系统
mkinitramfs -o /boot/initrd.img-`uname -r` `uname -r`
```


安装自编译驱动
---------

1. 


排错
---------


手把手超详细双系统安装教程
---------

某天没事做翻`bios`发现自己笔记本除了`AHCI`的模式，还支持`RAID`模式嘞，
奔着想要搞个`raid0`来提提速的想法，索性就来试一试这个传说中的笔记本`raid`模式。[^1]

<!-- source: https://kmpic.asus.com/images/2021/09/30/f1c03f6d-af06-4366-bb94-685e98528901.png -->
![asus-bios-sata-mode.png](pic/asus-bios-sata-mode.png)

根据`arch`教派的亲传[^2]，
这玩意可以通过`dmraid`来安装驱动来识别`raid`组，
不过这个貌似只支持牢英的`Intel Rapid Storage Technology`，
而且也被废弃了，目前牢英官方推荐使用`mdadm`进行软raid抽象层管理，
`dmraid`是通过`/dev/mapper`来管理设备的，而不是`/dev/sdx`，
下面不再使用`dmraid`安装，采用`AMD`官方的`rcraid`驱动管理。

### BIOS 配置

一般`DIY`主板用户在对应平台下载`raid驱动包`即可，
我是`天选4R`(AMD A+A YES)，故就在对应官网笔记本驱动下载。
详细过程非常简单，官方有一系列的操作图示，跟着就行。[^3]

> 1. 进入 BIOS 设定: 计算机重新启动时,在 POST\(开机自动测试\) 时按下 `F2`，进入BIOS 设定页面。
> 2. 当进入 BIOS 设定画面时, 将会出现计算机系统信息。之后在页面中按下 `F7` 进入 \[Advanced Mode\] \(进阶模式\)。
> 3. 在 \[Advanced Mode\] 设定页面中，选择 \[SATA Configuration\] 按下 `Enter` 进入。
> 4. 在 \[SATA Configuration\] 页面中，将 \[SATA Mode Selection\] 项目切换为 \[RAID\]，之后按下 `F10` 储存并离开BIOS设定，即完成开启 \[RAIDXpert2 Configuration Utility\]。
> 5. 重复1-2步骤进入BIOS 设定页面，选择 \[RAIDXpert2 Configuration Utility\] 按下 `Enter` 进入。
> 6. 在 \[RAIDXpert2 Configuration Utility\] 页面中，选择 \[Array Management\] 然后按下 `Enter`。
> 7. 在 \[Array Management\] 页面中，选择 \[Delete Array\] 项目后按下 `Enter`。
> 8. 在 \[Delete Array\] 页面中，将选定的RAID 数组切换至 \[On\] 或全选为 \[Check All\]，之后选择 \[Delete Array\(s\)\] 项目按下 `Enter`。
> 9. 最后在 \[Warning\] 页面中再次确认，完成后切换 \[Confirm\] 项目至 \[On\]，之后选择 \[Yes\] 开始执行删除。

<!-- source: https://kmpic.asus.com/images/2021/09/30/cba55582-2e90-4612-b653-f2b39726e3e6.png -->
![asus-bios-array.png](pic/asus-bios-array.png) 

在创建完`raid`阵列后，此处应该会出现阵列控制器的类型标识，
我这里的天选笔记本是`NVME DID`，
先记着，后面需要用到。

![rcraid-controller-mode](pic/rcraid-controller-mode.jpeg)


### 双系统 安装(Win侧)

#### 方案一：直接使用烧录的 Windows 安装镜像

使用镜像烧录到`优盘`或者`光盘`等`可移动媒介`触发微软官方的安装引导即可，
烧录之后还需要下载`rcraid`驱动，驱动在对应平台官网下载，
`rcraid`驱动大部分是通用的（
[官方下载](https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/RAID_Windows_Driver_932_00187.zip)或[备份下载](windows/WIN11/x64.tar.zst)
），下载之后解压放在`可移动媒介`上的任意位置。

> 注意: 备份下载源用的`tar(POSIX ustar)`打包，`Zstandard`压缩算法，
>		可能需要自行下载解压工具！

> 注意: 一般使用9.x版本的`rcraid`驱动，8.x版本的驱动只支持`SATA`。
> 		[官方手册](pdf/53987-3.12.pdf#page=25)第25页已列出详细说明[^4]
> 
> See the system’s motherboard specifications for the supported device interface. RC-
> 8.1 and RC-8.2 branches support only SATA. RC-9.1 and above release branches
> support SATA, M.2 NVMe SSD and HHHL NVMe SSD


在`选择安装Windows的位置`界面正常应该显示多个无法识别的`Linux Swap`分区，
此处选择`加载驱动程序`加载刚刚下载解压后的驱动文件夹，
根据自己`阵列类型`选择对应驱动。

![win-install](pic/win-install.png)

`阵列类型`在[官方手册](pdf/53987-3.12.pdf#page=55)第55页有详细说明。
在上方教程`bios 配置`中可以看到自己`阵列类型`。[^4]

一般来说SATA接口都是`RAID_SATA`, M.2 key E接口是`NVMe_DID`或`NVMe_CC`，
`CC`是老设备兼容,`DID`是新M2驱动格式，
一般3000系带集显的桌面端锐龙处理器、二代以上线程撕裂者处理器、移动设备处理器选`DID`，
其余`NVMe`都是`CC`。

选择之前下载解压后的驱动，没问题的话应该是导入成功三个驱动(`rcbottom,rccfg,rcraid`)。

![select-rcraid](pic/select-rcraid.png)

> 注意: 不要把`DID CC SATA`都导入了，只能导入其中一个。

导入正常后返回`选择安装Windows的位置`界面，
按照正常方式选择阵列磁盘安装系统即可。

进入安装好的系统做完`OOBE`即可完成。

#### 方案二：使用 WinPE 10+ DISM/Ghost 安装

先决条件: 一个可以使用的Windows PE或者RE，注意需要Windows NT 10以上的内核。

下载`rcraid`驱动，驱动在对应平台官网下载，
`rcraid`驱动大部分是通用的（
[官方下载](https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/RAID_Windows_Driver_932_00187.zip)或[备份下载](windows/WIN11/x64.tar.zst)
），下载之后解压放在PE/RE能够访问的任意位置。

> 注意: 备份下载源用的`tar(POSIX ustar)`打包，`Zstandard`压缩算法，
>		可能需要自行下载解压工具！

> 注意: 一般使用9.x版本的`rcraid`驱动，8.x版本的驱动只支持`SATA`。
> 		[官方手册](pdf/53987-3.12.pdf#page=25)第25页已列出详细说明[^4]
> 
> See the system’s motherboard specifications for the supported device interface. RC-
> 8.1 and RC-8.2 branches support only SATA. RC-9.1 and above release branches
> support SATA, M.2 NVMe SSD and HHHL NVMe SSD

进入`PE`后，`Win+X`选择`设备管理器`进入，点击`添加驱动程序`。

![win-dev-mgr](pic/win-dev-mgr.png)

`阵列类型`在[官方手册](pdf/53987-3.12.pdf#page=55)第55页有详细说明。
在上方教程`bios 配置`中可以看到自己`阵列类型`[^4]

一般来说SATA接口都是`RAID_SATA`, M.2 key E接口是`NVMe_DID`或`NVMe_CC`，
`CC`是老设备兼容,`DID`是新M2驱动格式，
一般3000系带集显的桌面端锐龙处理器、二代以上线程撕裂者处理器、移动设备处理器选`DID`，
其余`NVMe`都是`CC`。

选择之前下载解压后的驱动，没问题的话应该是导入成功三个驱动(`rcbottom,rccfg,rcraid`)。

![select-rcraid](pic/select-rcraid.png)

> 注意: 不要把`DID CC SATA`都导入了，只能导入其中一个。

导入成功后，选择对应阵列磁盘，按照正常方式安装即可，
`dism`、`ghost`安装方法都没任何问题。
安装完毕之后还需要给安装的系统打上`rcraid`驱动。

使用`ghost`（已废弃）：

- `.gho`恢复镜像前后自行选择`dism`安装驱动。

使用`dism`（不推荐）：

- 给`wim/esd/vhd/vhdx/vfd`等未安装的镜像打入驱动,
此处的路径需要根据你的需要自行修改
```powershell
# 确定镜像文件和索引号(wim可改为esd/vhd/vhx/vfd等等,后面同理)
Dism /Get-ImageInfo /ImageFile:"C:\path\to\install.wim"
# 挂载空挂载点
mkdir D:\mount
# 挂载 Windows 镜像
Dism /Mount-Image /ImageFile:"C:\path\to\install.wim" /Index:你的索引号 /MountDir:"D:\mount"
# 添加驱动程序到挂载的镜像(dism自动搜索)
Dism /Image:"D:\mount" /Add-Driver /Driver:"C:\path\to\your\driver\folder" /Recurse
# 卸载镜像并保存更改
Dism /Unmount-Image /MountDir:"D:\mount" /Commit
```

- 给安装完毕的系统打入驱动
```powershell
# 选择系统盘打入驱动
Dism /Image:"D:\" /Add-Driver /Driver:"C:\path\to\your\driver\folder" /Recurse
```

使用图形化`dism++`（推荐）：

- 释放镜像安装系统

![dismplusplus-iso](pic/dismplusplus-iso.png)

选择`安装的系统`>选择`驱动管理`>选择`添加驱动`\
选择之前下载解压后的驱动，没问题的话应该是导入成功三个驱动(`rcbottom,rccfg,rcraid`)\
![dismplusplus-install](pic/dismplusplus-install.png)

进入安装好的系统做完`OOBE`即可完成。


### 双系统 安装(Linux侧)

下载驱动


### 安装管理器

#### Windows侧

使用 `Microsoft Store` 或者 `winget` 下载。[^5]

```powershell
winget install -e --id 9nd45c02n872
```

也可使用 `第三方安装包` 侧载。[^6]

进入[`store.rg-adguard.net`](https://store.rg-adguard.net/)或[`备用下载`](windows/manager/UWP_Certified.tar.zst)，
选择`ProductID`，输入`9nd45c02n872`，选择`RP`，确定。
先下载安装对应架构的`VCLibs`依赖包，再下载安装本体。

#### Linux侧





```plaintext
下载仓库来源
linux
 ├─KERNEL6+: manully patch
 ├─manager: https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/RAIDXpert2_Linux_932_00230.zip
 └─UBUNTU22.04.1: https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/RAID_Linux_Ubuntu_2.2.40-9.3.2.00230.zip
windows
 ├─manager: https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/RAIDXpert2_Windows_932_00195.zip
 ├─ROG: https://dlcdnet.asus.com/pub/ASUS/GamingNB/DriverforWin10/Raid_ROG/Raid_windows_driver_930_00266.zip
 ├─WIN10: https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/Raid_PKG_S0i3_win_9.3.2_00158.zip
 └─WIN11: https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/RAID_Windows_Driver_932_00187.zip
```

[^1]: 幻16逆天跑分：https://www.bilibili.com/video/BV17W4y1h7mT
[^2]: fakeraid介绍：https://wiki.archlinuxcn.org/wiki/RAID#实现方式
[^3]: 华硕笔记本raid教程：https://www.asus.com.cn/support/faq/1046579/#cc
[^4]: 官方文档53987 3.12修订版: https://drivers.amd.com/relnotes/amd-raidxpert2_user_guide_3.12.pdf
[^5]: AMD RAIDXpert2 windows官网下载地址: https://apps.microsoft.com/detail/9nd45c02n872
[^6]: AMD RAIDXpert2 windows厂商下载地址: https://www.supermicro.com/wdl/driver/AMD/NVMe_RAID/RAIDXpert2_Windows_932_00195.zip
