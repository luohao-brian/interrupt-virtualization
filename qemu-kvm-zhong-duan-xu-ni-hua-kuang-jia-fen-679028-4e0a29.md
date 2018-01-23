本小结我们来聊一聊中断虚拟化，结合qemu-kvm虚拟化原理来分析下中断虚拟化框架。

> 中断虚拟化的关键在于对中断控制器的模拟，我们知道x86上中断控制器主要有旧的中断控制器PIC\(intel 8259a\)和适应于SMP框架的IOAPIC/LAPIC两种。

### 1. 中断控制器的创建和初始化

考虑到中断实时性对性能的影响，PIC和IOAPIC的设备模拟主要逻辑都放到了kvm模块进行实现，每个VCPU的LAPIC则完全放到kvm中进行实现。 i8259控制器和IOAPIC的创建和初始化由qemu和kvm配合完成，包括了kvm中设备相关数据结构初始化和qemu中设备模拟的初始化2个方面：

#### \(1\)中断控制器的创建

qemu代码中中断控制器的kvm内核初始化流程为：

```
configure_accelerator
    |--> accel_init_machine
        |--> kvm_init
            |--> kvm_irqchip_create
                |--> kvm_vm_ioctl(s, KVM_CREATE_IRQCHIP)
                |--> kvm_init_irq_routing
```

qemu通过kvm的ioctl命令KVM\_CREATE\_IRQCHIP调用到kvm内核模块中，在内核模块中创建和初始化PIC/IOAPIC设备（创建设备对应的数据结构并将设备注册到总线上）。

```
kvm_arch_vm_ioctl(s, KVM_CREATE_IRQCHIP)
    |--> kvm_pic_init                    /* i8259 初始化 */
    |--> kvm_ioapic_init                 /* ioapic 初始化 */
    |--> kvm_setup_default_irq_routing   /* 初始化缺省的IRE */
```

qemu在kvm内核中创建完成PIC和IOAPIC后将全局变量_kvm\_kernel\_irqchip_置为true，kvm模块则将kvm-&gt;arch.irqchip\_mode 赋值为 KVM\_IRQCHIP\_KERNEL，这样后面的kvm\_irqchip\_in\_kernel返回true表示pic芯片放到kvm内核模块中实现,kvm\_ioapic\_in\_kernel也返回true表示ioapic放到kvm中来模拟。

中断处理的逻辑放在kvm内核模块中进行实现，但设备的模拟呈现还是需要qemu设备模拟器来搞定，最后qemu和kvm一起配合完成快速中断处理的流程。

i8259的设备创建流程\(pic还是传统的isa设备，中断是边沿触发的，master的i/o port为0x20,0x21 slave的i/o port为0xa0,0xa1\)：

```
machine_run_board_init
    |--> pc_init1
        |--> if (kvm_pic_in_kernel())
            |--> kvm_i8259_init
                |--> isadev = isa_create(bus, name)
```

ioapic的设备创建流程:

```
machine_run_board_init
    |--> pc_init1
        |--> if (pcmc->pci_enabled)
            |--> ioapic_init_gsi(gsi_state, "i440fx")
                |--> if kvm_ioapic_in_kernel()
                    |--> dev = qdev_create(NULL, "kvm-ioapic")
```

PIC由2个i8259进行“级联”，一个为master一个为slave，每个i8259有8个PIN（salve的INT输出线连接到master的IRQ2引脚上,所以实际可用的IRQ数目为15）。目前kvm只为虚拟机创建一个ioapic设备（现在多路服务器可能有多个ioapic设备），ioapic设备提供24个PIN给外部中断使用。在IRQ路由上 0-15号GSI为PIC和IOAPIC共用的，16-23号GSI则都分配给ioapic。

几个概念要理清楚：IRQ号，中断向量和GSI。

* IRQ号是PIC时代引入的概念,由于ISA设备通常是直接连接到到固定的引脚，所以对于IRQ号描述了设备连接到了PIC的哪个引脚上，同IRQ号直接和中断优先级相关,例如IRQ0比IRQ3的中断优先级更高。
* GSI号是ACPI引入的概念，全称是Global System Interrupt，用于为系统中每个中断源指定一个唯一的中断编号。注：ACPI Spec规定PIC的IRQ号必须对应到GSI0-GSI15上。kvm默认支持最大1024个GSI。
* 中断向量是针对逻辑CPU的概念，用来表示中断在IDT表的索引号，每个IRQ（或者GSI）最后都会被定向到某个Vecotor上。对于PIC上的中断，中断向量 = 32\(start vector\) + IRQ号。在IOAPIC上的中断被分配的中断向量则是由操作系统分配。

PIC主要针对与传统的单核处理器体系结构，在SMP系统上则是通过IOAPIC和每个CPU内部的LAPIC来构成整个中断系统的。

![](https://kernelgo.org/images/ioapic.png "ioapic")

如上图所描述，IOAPIC 负责接受中断并将中断格式化化成中断消息，并按照一定规则转发给LAPIC。LAPIC内部有IRR\(Interrupt Reguest Register\)和ISR\(Interrupt Service Register\)等2个重要寄存器。系统在处理一个vector的同时缓存着一个相同的vector，vector通过2个256-bit寄存器标志，对应位置位则表示上报了vector请求或者正在处理中。另外LAPIC提供了TPR\(Task Priority Register\)，PPR\(Processor Priority Register\)来设置LAPIC的task优先级和CPU的优先级，当IOAPIC转发的终端vector优先级小于LAPIC设置的TPR时，此中断不能打断当前cpu上运行的task；当中断vector的优先级小于LAPIC设置的PPR时此cpu不处理这个中断。操作系统通过动态设置TPR和PPR来实现系统的实时性需求和中断负载均衡。

值得一提的是qemu中为了记录pic和ioapic的中断处理回调函数，定义了一个GSIState类型的结构体：

```
typedef struct GSIState {
    qemu_irq i8259_irq[ISA_NUM_IRQS];
    qemu_irq ioapic_irq[IOAPIC_NUM_PINS];
} GSIState;
```

在qemu主板初始化逻辑函数pc\_init1中会分别分配ioapic和pic的qemu\_irq并初始化注册handler。ioapic注册的handler为kvm\_pc\_gsi\_handler函数opaque参数为qdev\_get\_gpio\_in,pic注册的handler为kvm\_pic\_set\_irq。这2个handler是qemu模拟中断的关键入口，后面我们会对其进行分析。

#### \(2\) 用户态和内核态的中断关联

qemu中尽管对中断控制器进行了模拟，但只是搭建了一个空架子，如果高效快速工作起来还需要qemu用户态和kvm内核的数据关联才能实现整个高效的中断框架。

IOAPIC为了实现中断路由\(Interrupt Routing\)会维护一个中断路由表信息，下面看下kvm内核模块中几个重要的数据结构。

中断路由：用来记录中断路由信息的数据结构。

```
struct kvm_irq_routing {
    __u32 nr;
    __u32 flags;
    struct kvm_irq_routing_entry entries[0];  /* irq routing entry 数组 */
};
```

中断路由表：

kvm\_irq\_routing\_table这个数据结构描述了“每个虚拟机的中断路由表”，对应于kvm数据结构的irq\_routing成员。chip是个二维数组表示三个中断控制器芯片的每一个管脚（最多24个pin）的GSI，nr\_rt\_entries表示中断路由表中存放的“中断路由项”的数目，最为关键的struct hlist\_head map\[0\]是一个哈希链表结构体数组，数组以GSI作为索引可以找到同一个irq关联的所有kvm\_kernel\_irq\_routing\_entry（中断路由项）。

```
struct kvm_irq_routing_table {
    int chip[KVM_NR_IRQCHIPS][KVM_IRQCHIP_NUM_PINS];
    u32 nr_rt_entries;
    /*
        * Array indexed by gsi. Each entry contains list of irq chips
        * the gsi is connected to.
        */
    struct hlist_head map[0];  /* 哈希表数组 */
};
```

中断路由项：

gsi表示这个中断路由项对应的GSI号，type表示该gsi的类型取值可以是 KVM\_IRQ\_ROUTING\_IRQCHIP, KVM\_IRQ\_ROUTING\_MSI等，set函数指针很重要表示该gsi关联的中断触发方法（不同type的GSI会调用不同的set触发函数），hlist\_node则是中断路由表哈希链表的节点，通过link将同一个gsi对应的中断路由项链接到map对应的gsi上。

```
struct kvm_kernel_irq_routing_entry {
    u32 gsi;
    u32 type;
    int (*set)(struct kvm_
    kernel_irq_routing_entry *e,
        struct kvm *kvm, int irq_source_id, int level,
        bool line_status);
    union {
        struct {
            unsigned irqchip;
            unsigned pin;
        } irqchip;
        struct {
            u32 address_lo;
            u32 address_hi;
            u32 data;
            u32 flags;
            u32 devid;
        } msi;
        struct kvm_s390_adapter_int adapter;
        struct kvm_hv_sint hv_sint;
    };
    struct hlist_node link;
};
```

中断路由表的设置在qemu中初始化时，通过调用kvm\_pc\_setup\_irq\_routing函数来完成。

```
    void kvm_pc_setup_irq_routing(bool pci_enabled)
    {
        KVMState *s = kvm_state;
        int i;

        if (kvm_check_extension(s, KVM_CAP_IRQ_ROUTING)) {
            for (i = 0; i < 8; ++i) {
                if (i == 2) {    /* slave的INTR引脚接入到master的2号引脚上 */
                    continue;
                }
                kvm_irqchip_add_irq_route(s, i, KVM_IRQCHIP_PIC_MASTER, i);
            }
            for (i = 8; i < 16; ++i) {
                kvm_irqchip_add_irq_route(s, i, KVM_IRQCHIP_PIC_SLAVE, i - 8);
            }
            if (pci_enabled) {
                for (i = 0; i < 24; ++i) {
                    if (i == 0) {
                        kvm_irqchip_add_irq_route(s, i, KVM_IRQCHIP_IOAPIC, 2);
                    } else if (i != 2) {
                        kvm_irqchip_add_irq_route(s, i, KVM_IRQCHIP_IOAPIC, i);
                    }
                }
            }
            kvm_irqchip_commit_routes(s);
        }
    }
```



