> 一个操作系统要跑起来，必须有Time Tick，它就像是身体的脉搏。普通情况下，OS Time Tick由PIT\(i8254\)或APIC Timer设备提供—PIT定期\(1ms in Linux\)产生一个timer interrupt，作为global tick, APIC Timer产生一个local tick。在虚拟化情况下，必须为guest OS模拟一个PIT和APIC Timer。  
> 模拟的PIT和APIC Timer不能像真正硬件那样物理计时，所以一般用HOST的某种系统服务或软件计时器来为这个模拟PIT提供模拟”时钟源”。

目前两种方案：1. 用户态模拟方案（QEMU）； 2. 内核态模拟方案（KVM）；  
在QEMU中，用SIGALARM信号来实现：QEMU利用某种机制，使timer interrupt handler会向QEMU process发送一个SIGALARM信号，处理该信号过程中再模拟PIT中产生一次时钟。QEMU再通过某种机制，将此模拟PIT发出的模拟中断交付给kvm，再由kvm注入到虚拟机中去。  
目前的kvm版本支持内核PIT、APIC和内核PIC，因为这两个设备是频繁使用的，在内核模式中模拟比在用户模式模拟性能更高。这里重点是讲内核PIT的模拟实现，弄清楚它是如何为guest OS提供时钟的。

## 相关代码 {#相关代码}

主要涉及本主题讨论的代码为：

```
linux/arch/x86/kvm/i8254.c
linux/arch/x86/kvm/i8254.h
linux/arch/x86/kvm/i8259.c
linux/arch/x86/kvm/irq.c
linux/arch/x86/kvm/irq.h
linux/virt/kvm/irq_comm.c
```

## 物理芯片介绍 {#物理芯片介绍}

1. PIT 主要为Intel 8254 PIT芯片；

   PIT \(Programmable Interval Timer\) 可编程间隔定时器

   每个PC机中都有一个PIT，通过IRQ产生周期性的时钟中断信号来充当系统定时器。

   i386中使用的通常是Intel 8254 PIT芯片，它的I/O端口地址范围是40h～43h。

   8254 PIT有3个计时通道，每个通道都有其不同的用途：

   通道0用来负责更新系统时钟。它在每一个时钟滴答会通过IRQ0向系统发出一次时钟中断信号。

   通道1通常用于控制DMAC对RAM的刷新。

   通道2被连接到PC机的扬声器，以产生方波信号。

2. PIC主要为8259A PIC芯片；

   PIC\(Programmable Interrupt Controller\) 可编程中断控制器

   它具有IR0~IR7共8个中断管脚连接外部设备。中断管脚具有优先级，其中IR0优先级最高，IR7最低。PIC有三个重要的寄存器：

   （1） IRR Interrupt Request Register, 中断请求寄存器 共8位，对应IR0~IR7这8个中断管脚。某位置1代表收到对应管脚的中断但还未提交给CPU。

   （2） ISR In Service Register 服务中寄存器：共8位，某位置1代表对应管脚的中断已经提交给CPU处理，但CPU还未处理完。

   （3） IMR Interrupt Mask Register 中断屏蔽寄存器：共8位，某位置1对应的中断管脚被屏蔽。

## 实现 {#实现}

### 创建流程 {#创建流程}

QEMU中创建入口：

```
kvm_vm_ioctl(s, KVM_CREATE_IRQCHIP)    (kvm-all.c)
kvm_vm_ioctl(kvm_state, KVM_CREATE_PIT)（i8254.c）
```

### 虚拟PIC的创建 {#虚拟PIC的创建}

首先看KVM\_CREATE\_IRQCHIP：  
对应内核代码为：kvm\_arch\_vm\_ioctl（） case KVM\_CREATE\_IRQCHIP

#### 第一步： {#第一步：}

vpic = kvm\_create\_pic\(\)

1. s = kzalloc\(sizeof\(struct kvm\_pic\), GFP\_KERNEL\); 创建kvm\_pic结构；
2. s-&gt;irq\_request = pic\_irq\_request; 注册pic\_irq\_request中断请求函数；
3. kvm\_iodevice\_init\(&s-&gt;dev, &picdev\_ops\); 注册PIC dev的IO端口读写函数；

```
static const struct kvm_io_device_ops picdev_ops = {
	.read     = picdev_read,
	.write    = picdev_write,
};
```

读写函数的进去第一句话即时判断读写地址是否属于本picdev的；picdev\_in\_range\(\)

1. kvm\_io\_bus\_register\_dev\(kvm, KVM\_PIO\_BUS, &s-&gt;dev\); 注册了PIO型的bus访问形式；另一种IO形式为MMIO；

        kvm-&gt;arch.vpic = vpic; 返回值vpic赋给了kvm的arch.vpic结构；

#### 第二步： {#第二步：}

r = kvm\_ioapic\_init\(kvm\); 初始化ioapic  
ioapic使用的是kvm\_io\_bus\_register\_dev\(kvm, KVM\_MMIO\_BUS, &ioapic-&gt;dev\);

#### 第三步： {#第三步：}

r = kvm\_setup\_default\_irq\_routing\(kvm\); 设置默认的irq路由 （irq\_comm.c）  
Time Tick占用irq0  
安装中断路由函数主要在setup\_routing\_entry中

```
case KVM_IRQCHIP_PIC_MASTER:
			e->set = kvm_set_pic_irq; 相当于中断路由函数
			max_pin = 16;
			break;
```

虚拟PIC创建流程到此为止；

### 虚拟PIT的创建 {#虚拟PIT的创建}

#### 第一步： {#第一步：-1}

struct kvm\_pit \*pit = kzalloc\(\) 申请kvm\_pit结构,代表一个PIT外设；

#### 第二步： {#第二步：-1}

pit-&gt;irq\_source\_id = kvm\_request\_irq\_source\_id\(kvm\); 申请中断资源号  
kvm-&gt;arch.vpit = pit; 赋予kvm的vpit；  
pit-&gt;kvm = kvm;

#### 第三步： {#第三步：-1}

hrtimer\_init\(&pit\_state-&gt;pit\_timer.timer, CLOCK\_MONOTONIC, HRTIMER\_MODE\_ABS\); 高精度定时器初始化

#### 第四步： {#第四步：}

kvm\_register\_irq\_ack\_notifier\(kvm, &pit\_state-&gt;irq\_ack\_notifier\);  
注册该pit的irq ack notifier

#### 第五步： {#第五步：}

kvm\_pit\_reset\(pit\);——- &gt;pit\_load\_count\(\) ———- &gt; create\_pit\_timer\(\):  
pt-&gt;timer.function = kvm\_timer\_fn;  
hrtimer\_start\(&pt-&gt;timer, ktime\_add\_ns\(ktime\_get\(\), interval\),  
HRTIMER\_MODE\_ABS\);  
到这里：内核虚拟PIT实际是利用HOST的hrtimer实际时钟源来提供虚拟时钟的。 Hrtimer为 高精度定时器，管理机制为:传统的低精度的为时间轮方案； 高精度方案为红黑树管理方案；此处注册的时间为1ms一次，并为周期性的；注册的中断处理函数kvm\_timer\_fn;

#### 第六步： {#第六步：}

kvm\_io\_bus\_register\_dev\(kvm, KVM\_PIO\_BUS, &pit-&gt;dev\); 注册PIO型设备；  
至此PIT也创建完成；

