> Qemu为virtio设备分配了专门的pci设备ID，device IDs \(vendor ID 0x1AF4\) from 0x1000 through 0x10FF,而pci子系统中的厂商ID和设备ID就成为了virtio类型和厂商域的组成，所以PCI驱动是不需要知道virtio设备类型的真正含义，对于Kernel来说只是注册了一个struct virtio\_device，并挂载到了virtio bus类型总线上，并由virtio driver来驱动。&gt; 
>
> virtio设备对于Linux Kernel中的设备类型来说是作为pci设备被使用的，因此具有pci设备的所有属性，所以其也具备了PCI配置空间。



## PCI配置空间域 {#PCI配置空间域}

Linux在PCI设备启动初始化过程中通过IO mapping方式对配置空间的访问地址进行了映射。如下宏定义了每一个配置寄存器相对于PIC配置io基址的偏移。

而通过对如下的相关配置域的配置实现通用的Guest特性、队列、中断等操作。

```
file: /include/uapi/linux, line: 45
```

```
/* A 32-bit r/o bitmask of the features supported by the host */
#define VIRTIO_PCI_HOST_FEATURES	0
/* A 32-bit r/w bitmask of features activated by the guest */
#define VIRTIO_PCI_GUEST_FEATURES	4
/* A 32-bit r/w PFN for the currently selected queue */
#define VIRTIO_PCI_QUEUE_PFN	 8
/* A 16-bit r/o queue size for the currently selected queue */
#define VIRTIO_PCI_QUEUE_NUM	 12
/* A 16-bit r/w queue selector */
#define VIRTIO_PCI_QUEUE_SEL	 14
/* A 16-bit r/w queue notifier */
#define VIRTIO_PCI_QUEUE_NOTIFY	 16
/* An 8-bit device status register. */
#define VIRTIO_PCI_STATUS	 18
/* An 8-bit r/o interrupt status register. Reading the value will return the
 * current contents of the ISR and will also clear it. This is effectively
 * a read-and-acknowledge. */
#define VIRTIO_PCI_ISR	 19
/* The bit of the ISR which indicates a device configuration change. */
#define VIRTIO_PCI_ISR_CONFIG	 0x2
/* MSI-X registers: only enabled if MSI-X is enabled. */
/* A 16-bit vector for configuration changes. */
#define VIRTIO_MSI_CONFIG_VECTOR 20
/* A 16-bit vector for selected queue notifications. */
#define VIRTIO_MSI_QUEUE_VECTOR 22
/* Vector value used to disable MSI for queue */
#define VIRTIO_MSI_NO_VECTOR 0xffff
```

## Guest中的操作方式 {#Guest中的操作方式}

Guest中virtio驱动通过iowrite/ioread等操作，比如：iowrite16\(index, vp\_dev-&gt;ioaddr + VIRTIO\_PCI\_QUEUE\_SEL\); 与virtio设备进行交互，而该io操作会被VMM截获，从而转移至Host中进行处理。

而在QEMU中为支持virtio设备的模拟提供了virtio\_portio函数注册结构来响应guest的不同读写请求。

```
file:/hw/virtio-pci.c, line: 464
```

```
static const MemoryRegionPortio virtio_portio[] = {
    { 0, 0x10000, 1, .write = virtio_pci_config_writeb, },
    { 0, 0x10000, 2, .write = virtio_pci_config_writew, },
    { 0, 0x10000, 4, .write = virtio_pci_config_writel, },
    { 0, 0x10000, 1, .read = virtio_pci_config_readb, },
    { 0, 0x10000, 2, .read = virtio_pci_config_readw, },
    { 0, 0x10000, 4, .read = virtio_pci_config_readl, },
    PORTIO_END_OF_LIST()
};
```



