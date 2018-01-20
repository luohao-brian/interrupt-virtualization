# 0. 硬件条件

确定CPU是否具有Virtualization Function：

```
$ cat /proc/cpuinfo | grep vmx
```

若有输出则表示该CPU支持Virtualization。  
同时在BIOS中将虚拟化支持选项打开。

## 1. Guest OS-Linux安装 {#1-_Guest_OS-Linux安装}

### 1.1 Host Linux安装 {#1-1_Host_Linux安装}

Ubuntu官网下载最新的iso文件，最新版本为Ubuntu 14.04 LTS\(链接：[http://www.ubuntu.com](http://www.ubuntu.com/)\)。  
得到ubuntu-14.04-desktop-amd64.iso，使用烧录工具使用U盘或者光盘进行安装。  
此过程若不会请自行度娘。

### 1.2 Qemu-kvm安装 {#1-2_Qemu-kvm安装}

安装完Ubuntu后，进行QEMU-KVM（相当于QEMU/KVM用户态工具）安装，可参考官网指导:  
[http://www.linux-kvm.org/page/HOWTO1](http://www.linux-kvm.org/page/HOWTO1)。

* 预安装相关开发包

```
$sudo apt-get install gcc libsdl1.2-dev zlib1g-dev libasound2-dev      linux-kernel-headers pkg-config libgnutls-dev libpci-dev
```

* 下载qemu-kvm-release.tar.gz

  [http://sourceforge.net/projects/kvm/files/](http://sourceforge.net/projects/kvm/files/)下载最新版本 qemu-kvm-1.2.0.tar.gz

![](http://7j1zbu.com1.z0.glb.clouddn.com/1.png)

* 对下载的qemu-kvm软件包编译安装

```
$tar zxvf qemu-kvm-1.2.0.tar.gz $cd qemu-kvm-1.2.0
```

注意：该qemu-kvm版本在Ubunut14.04 上是无法编过的，需要打一个Patch，Patch如下，可根据diff对configure文件进行修改。  
![](http://7j1zbu.com1.z0.glb.clouddn.com/2.png)

```
$./configure –prefix=/usr/local/kvm     //表示qemu-kvm工具安装目录 $make $sudo make install
```

安装完后会在/usr/local/kvm/bin目录下存在qemu-img/qemu-system-x86\_64等工具,至此，用户态工具安装完成。

### 1.3 KVM内核模块安装 {#1-3_KVM内核模块安装}

KVM内核模块包括：kvm.ko、kvm-intel.ko，可以先：

```
$lsmod |grep kvm
```

若有kvm.ko及kvm-intel.ko相关输出，则说明已经安装了kvm内核模块。否则进行安装

```
$lsmod |grep kvm
```

### 1.4 Guset OS安装 {#1-4_Guset_OS安装}

* 创建虚拟disk

![](http://7j1zbu.com1.z0.glb.clouddn.com/3.png)

```
$/usr/local/kvm/bin/qemu-img create -f qcow2 vdisk_linux.img 10G
```

参数说明：

```
1.create意思是创建一个新的磁盘；
2.-f指定该磁盘的类型这里选择qcow2,qemu 类型；
3.vdisk_linux.img为虚拟磁盘的文件名，然后就是磁盘初始大小，一般5G，10G；
```

* 启动Guest OS安装

```
$sudo /usr/local/kvm/bin/qemu-system-x86_64 -hda vdisk_linux.img      –cdrom ubuntu-14.04-desktop-amd64.iso -boot d -m 1024
```

参数说明：

```
1.-hda指定了硬盘是那个虚拟磁盘，这里用刚刚创建的vdisk_linux.img；

2.-cdrom指定启动的cdrom，可以用iso文件，也可以用机器的光驱，我们选择用iso文件;
   如果用光驱尝试-cdrom /dev/cdrom；

3.-boot指定XP启动的时候从磁盘，硬盘，光驱启动，我们安装的时候选择从光盘启动，所以用d；

4.-m虚拟机使用的内存大小，单位是MB,这里用1024MB（1GB）；
```

![](http://7j1zbu.com1.z0.glb.clouddn.com/4.png)

这样就启动Ubuntu的安装;

![](http://7j1zbu.com1.z0.glb.clouddn.com/5.png)

安装过程与在host machine上一样;

![](http://7j1zbu.com1.z0.glb.clouddn.com/7.png)

* Guest OS的启动

安装完成之后，重启进入已安装的系统：

```
$sudo /usr/local/kvm/bin/qemu-system-x86_64 -hda vdisk_linux.img      –cdrom ubuntu-14.04-desktop-amd64.iso -boot d -m 1024
```

此时相对于安装时删掉–cdrom ubuntu-14.04-desktop-amd64.iso.iso让其从硬盘启动

![](http://7j1zbu.com1.z0.glb.clouddn.com/8.png)

登陆进入Guest OS后：

![](http://7j1zbu.com1.z0.glb.clouddn.com/9.png)

可以看到终端的显示有点失真，可能是显卡驱动没有配置，后续再另行解决。

## 2. Guest OS-Windows安装 {#2-_Guest_OS-Windows安装}

既然KVM可以装Linux，那么咱们试试装windows。一样的步骤:

* 创建虚拟磁盘

```
$/usr/local/kvm/bin/qemu-img create -f qcow2 vdisk_win7.img 10G
```

* 启动iso的安装

```
$sudo /usr/local/kvm/bin/qemu-system-x86_64 -hda vdisk_win7.img      –cdrom win7_64_cn.iso -boot d -m 1024
```

![](http://7j1zbu.com1.z0.glb.clouddn.com/10.png)

![](http://7j1zbu.com1.z0.glb.clouddn.com/11.png)

![](http://7j1zbu.com1.z0.glb.clouddn.com/12.png)

* 安装完启动

```
$sudo /usr/local/kvm/bin/qemu-system-x86_64 -hda vdisk_win7.img  -boot d -m 1024
```

![](http://7j1zbu.com1.z0.glb.clouddn.com/13.png)

内存1GB：

![](http://7j1zbu.com1.z0.glb.clouddn.com/14.png)

硬盘10G，刚刚够…

![](http://7j1zbu.com1.z0.glb.clouddn.com/15.png)

