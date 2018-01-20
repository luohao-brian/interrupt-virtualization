> 相信很多的人对虚拟机并不陌生，目前也有很多优秀的虚拟机软件，例如：VMware, VirtualBox, Xen, KVM等。而本文的主要内容是介绍KVM。

## KVM: Kernel Based Virtual Machine {#KVM:_Kernel_Based_Virtual_Machine}

援引[KVM官网](http://www.linux-kvm.org/)对KVM的介绍:

> KVM \(for Kernel-based Virtual Machine\) is a full virtualization solution for Linux on x86 hardware containing virtualization extensions \(Intel VT or AMD-V\).It consists of a loadable kernel module, kvm.ko, that provides the core virtualization infrastructure and a processor specific module, kvm-intel.ko or kvm-amd.ko.

解释如下：

* KVM（内核虚拟机）是一个为Linux针对具有虚拟化扩展功能\(Intel 的VT技术、AMD的AMD-V技术\)的x86平台提供全虚拟化解决方案的软件；
* 由可加载的模块组成：kvm.ko：提供虚拟化的核心基础架构； kvm-intel.ko/kvm-amd.ko:处理器体系架构相关的模块；
* 使用QEMU工作；
* 能够运行多个异构的虚拟OS；
* 2.6.20+版本具有KVM特性；
* 开源；

## VT-x {#VT-x}

以Intel的硬件辅助虚拟化技术为例：Hardware Assisted Virtualization Intel Virtualization Technology。

## 背景 {#背景}

* Intel MicroProcessors 提供2-bit的特权级描述：0：最高特权级；3：最低特权级。
 
  大多数的软件只用了其0和3级，OS运行于0级，应用程序运行于1级，如下图所示：

![](http://7jpssh.com1.z0.glb.clouddn.com/1.PNG)

* 对于需要控制CPU的OS来讲，其组件需要运行在特权级0上，但对于VMM\(Virtual Machine Monitor\)来说，其是不允许Guest OS运行于最高特权级0。
 
  因此对于基于Intel体系结构的VMM,必须使用Ring Deprivileging技术，即Guest软件运行于比0级高的特权级上，级\(0/1/3\)\(0/3/3\)模式。如下图所示：

![](http://7jpssh.com1.z0.glb.clouddn.com/2.PNG)![](http://7jpssh.com1.z0.glb.clouddn.com/3.PNG)

* Ring Deprivileging技术引入的问题：

  * Ring Aliasing
  * Address space compression
  * Nonfaulting accessing to privileged state
  * Adverse impact on guest transitions
  * Interrupt virtualization
  * Access to hidden state

上述问题详细原因和分析可参看“Hardware Assisted Virtualization Intel Virtualization Technology”一文。

总之，虽然采用ring deprivileging方法可能实现系统虚拟化，但具有很多缺陷，且软件上比较复杂。为此，Intel提出了VT-x技术来解决系统虚拟化问题。

## 解决方案 {#解决方案}

### **新的模式** {#新的模式}

VT-x技术引入了两种模式，称为：

* 根操作模式\(VMX Root Operation\)
* 非根操作模式\(VMX Non-Root Operation\)

其中VMM运行在根操作模式下，而Guest OS则运行在非根操作模式下。  
每个模式都存在Ring0-3四个特权级别，这样其实就是CPU将OS的运行环境分为了两个世界，而每个世界中都有0-3的特权级，这样可以和原有非虚拟化运行模式相兼容。如下图所示：

![](http://7jpssh.com1.z0.glb.clouddn.com/4.PNG)

### **新的指令** {#新的指令}

* VMX ON/VMX OFF: 用来打开和关闭VMX操作模式的指令，在默认情况下，VMX是关闭的，当需要使用这个功能时，可以通过VMX ON随时进行VMX模式;
* VMLAUNCH/VMRESUME: 在进入VMX模式后，VMM又会通过VMLAUNCH或者VM RESUME指令产生VMEntry，使CPU从根操作模式切换至非根操作模式，从而开始运行Guest相关软件。
* 在运行软件的过程中如果发生中断或者异常，就会激活VM-Exit操作，此时CPU又进行了一次模式切换，只不过这次是切换到根操作模式，在处理完成后一般又会返回非根操作模式去运行Guest软件。

### **新的数据结构** {#新的数据结构}

VMCS是Intel-x中一个很重要的数据结构，它占用一个page大小，由VMM分配，但是硬件是需要读写的，有点类似于页表。  
VMCS数据域包括了6大类信息：

1. CPU状态。当VM-Exit发生时，CPU把当前状态存入客户机状态域；当VM-Entry发生时，CPU从客户机状态域恢复状态；
2. VMM运行时，即根模式时的CPU状态；当VM-Exit发生时，CPU从该域恢复CPU状态；
3. VM-EntryVM-Entry的过程；
4. VM-ExecutionVMX非根模式下的行为；
5. VM-ExitVM-Exit的过程；
6. VM-Exit信息域：提供VM-Exit原因和其他的信息

## KVM提供的相关API {#KVM提供的相关API}

KVM提供的相关API可以通过内核源码的Document查看:/document/virtual/kvm/api.txt;  
这里简单介绍下涉及到KVM虚拟机创建的三大文件描述符。

### _KVM: kvmfd_ {#KVM:_kvmfd}

kvm内核模块本身是作为一个设备驱动程序安装的，即kvm.ko，驱动的设备名称是/dev/kvm。  
要使用kvm，需要先open “/dev/kvm”设备，得到一个kvm设备文件描述符fd，然后利用此fd调用ioctl就可以向KVM设备发送命令了。  
QEMU中的open的地方为kvm\_init\(kvm-all.c\)。  
内核kvm模块解析此种请求的函数是kvm\_dev\_ioctl\(kvm\_main.c\)，如:KVM\_GET\_API\_VERSION/KVM\_CREATE\_VM。

### _VM: vmfd_ {#VM:_vmfd}

通过KVM\_CREATE\_VM创建了一个VM后，用户程序需要发送一些命令给VM，如:KVM\_CREATE\_VCPU。这些命令当然也是要通过ioctl来发送，所以VM也需要对应一个文件描述符才行。  
用户程序中用ioctl发送KVM\_CREATE\_VM得到的返回值就是新创建VM对应的fd，之后利用此fd发送命令给此VM。  
每个具体的VM都有一个struct kvm与之对应；内核kvm模块解析此种请求的函数是kvm\_vm\_ioctl\(kvm\_main.c\)。

### _VCPU: vcpufd_ {#VCPU:_vcpufd}

原理基本跟VM情况差不多，内核kvm模块解析此种请求的函数是kvm\_vcpu\_ioctl\(kvm\_main.c\)。VCPU控制块结构为struct kvm\_vcpu。

