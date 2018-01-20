> 上文针对Qemu在前端网络流路径的建立方面做了详细的描述，数据包从Host的物理网卡经过Host Linux内核中的Bridge， 经过Tap设备到达了Qemu的用户态空间。而Qemu是如何把数据包送进Guest中的呢，这里必然要说到到虚拟网卡的建立。

当命令行传入nic相关参数时，Qemu就会解析网络相关的参数后进入虚拟网卡的创建流程。而在上文中提到对于所有-net类型的设备，都视作一个net client来对待。而在net client的建立之前，需要先创建Qemu内部的hub和对应的port，来关联每一个net client，而对于每个创建的-net类型的设备都是可以可以配置其接口的vlan号，从而控制数据包在其中配置的vlan内部进行转发，从而做到多个虚拟设备之间的switch。



## Hub及port的建立 {#Hub及port的建立}

```
main()  file: vl.c, line: 2345
        net_init_clients()  file: net.c, line: 991
            net_init_client()  file: net.c, line: 962
                net_client_init()  file: net.c, line: 701
                    net_client_init1()  file: net.c, line: 628
                        peer = net_hub_add_port(u.net->has_vlan ? u.net->vlan : 0, NULL);
```

net\_hub\_add\_port\(\)传入的第一个参数为vlan号，如果没有配置vlan的话则默认为0。

```
NetClientState *net_hub_add_port(int hub_id, const char *name)
{
    NetHub *hub;
    NetHubPort *port;
    QLIST_FOREACH(hub, &hubs, next) {  <----从全局的hubs链中查找是否已经存在
        if (hub->id == hub_id) {
            break;
        }
    }
    if (!hub) {   
        hub = net_hub_new(hub_id);     <----若不存在则创建新的
    }
    port = net_hub_port_new(hub, name);<----创建在对应hub下创建新的port
    return &port->nc;
}
```

对于在此过程中创建的各种数据结构关系如下图：  
![](http://7kttfc.com1.z0.glb.clouddn.com/1.png)

相关的解释：

1. hubs全局链挂载的NetHub结构对应于指定创建的每一个vlan号；
2. 每个hub下面可以挂载属于同一vlan的多个NetHubPort来描述的port；
3. 每个NetHubPort下归属的NetClientState nc结构表示了具体挂载的net client;
4. “e1000”,”tap1”都有自己的net client结构，而各自的NetClientInfo都指向了net\_hub\_port\_info；该变量定义了一系列hub操作函数；
5. 最后net\_hub\_add\_port\(\)返回了每个nic对应的NetClientState结构，即图中的peer。

### nic设备与hub port的关联 {#nic设备与hub_port的关联}

nic设备完成hub及port的创建后，进入nic相关的初始化，即net\_init\_nic\(\)。  


```
staticintnet_init_nic(const NetClientOptions *opts, constchar *name, NetClientState *peer)
{
	NICInfo *nd;
	idx = nic_get_free_idx();
	nd = &nd_table[idx];
	nd->netdev = peer;
	nd->name = g_strdup(name);
	......
	nd->used = 1;
    nb_nics++;
return idx;
}
```

解释：

1. 首先从nd\_table\[\]中找到空闲的NICInfo结构，每一个nd\_table项代表了一个NIC设备；
2. 填充相关的内容，其中最重要的是net-&gt;netdev = peer，此时hub中的NetClient与NICInfo关联，通过NICInfo可找到NetClient；

![](http://7kttfc.com1.z0.glb.clouddn.com/2.png)

### tap设备与hub port的关联 {#tap设备与hub_port的关联}

tap设备完成hub及port的创建后，进入tap相关的初始化，即net\_init\_tap\(\)。前文中已描述过部分的tap设备相关初始化内容，主要是tap设备的打开和事件监听的设置。而这里主要描述hub port与tap设备的关联。

主要调用流程：

```
net_init_tap()  file: tap.c, line: 589
    net_tap_init()   file: tap.c, line: 552
        tap_open()   file: tap-linux.c, line: 38
            fd = open(PATH_NET_TUN, O_RDWR)   file: tap-linux.c, line: 43
    net_tap_fd_init()   file: tap.c, line: 325
static TAPState *net_tap_fd_init(NetClientState *peer, const char *model, const char *name, int fd, int vnet_hdr)
{
    NetClientState *nc;
    TAPState *s;
    nc = qemu_new_net_client(&net_tap_info, peer, model, name);
    s = DO_UPCAST(TAPState, nc, nc);
    s->fd = fd;
    s->host_vnet_hdr_len = vnet_hdr ? sizeof(struct virtio_net_hdr) : 0;
    s->using_vnet_hdr = 0;
    s->has_ufo = tap_probe_has_ufo(s->fd);
    tap_set_offload(&s->nc, 0, 0, 0, 0, 0);
    tap_read_poll(s, 1);
    s->vhost_net = NULL;
    return s;
}
```

该函数最主要的工作就是建立了代表tap设备的TAPState结构与hub port的关联;Tap设备对应的hub port的NetClientState的peer指向了TAPState的NetClient，而TAPState的NetClient的peer指向了hub port的NetClientInfo；  
tap\_read\_poll\(\)就是上文分析的设置tap设备读写监听事件。

![](http://7kttfc.com1.z0.glb.clouddn.com/3.png)

## 虚拟网卡的建立 {#虚拟网卡的建立}

上面图中可以看到TAP设备结构与tap对应的hub port通过NetClientInfo的peer指针相互进行了关联，而代表e1000的NICInfo中的NetClientInfo与e1000对应的hub port只有单向的关联，从hub port到NICInfo并没有关联，因此建立两者相互关联需要说到Guest的虚拟网卡的建立。

### Guest OS物理内存管理 {#Guest_OS物理内存管理}

在说明虚拟网卡的建立前，首先得说一下Guest OS中的物理内存管理。  
在虚拟机创建之初，Qemu使用malloc\(\)从其进程地址空间中申请了一块与虚拟机的物理内存大小相等的区域，该块区域就是作为了Guest OS的物理内存来使用。  
物理内存通常是不连续的，例如地址0xA0000至0xFFFFF、0xE0000000至0xFFFFFFFF等通常留给BIOS ROM和MMIO,而不是物理内存。  
设:  
虚拟机包括n块物理内存，分别记做P1, P2, …, Pn;  
每块物理内存的起始地址分别记做PB1, PB2, …, PBn;  
每块物理内存的大小分别为PS1, PS2, …, PSn。

Qemu根据虚拟机的物理内存布局，将该区域划分成n个子区域，分别记做V1, V2, …, Vn;  
第i个子区域与第i块物理内存对应，每个子区域的起始线性地址记做VB1, VB2, …, VBn;  
每个子区域的大小等于对应的物理内存块的大小，仍是PS1, PS2, …, PSn。

在Qemu创建虚拟机的时候会向KVM通告Guest OS所使用的物理内存布局，采用KVMSlot的数据结构来表示：

```
typedef struct KVMSlot
{
     hwaddr start_addr;        <----------Guest物理地址块的起始地址
     ram_addr_t memory_size;   <----------大小
     void *ram;                <----------QUMU用户空间地址
     int slot;                 <----------slot号
     int flags;                <----------内存属性
} KVMSlot;
```

调用关系：

```
Qemu:
kvm_set_phys_mem()->            file:kvm-all.c, line:550
    kvm_set_user_memory_region()->     file:kvm-all.c, line:191
        kvm_vm_ioctl()->        通过KVM_SET_USER_MEMORY_REGION进入Kernel

KVM:
kvm_vm_ioctl_set_memory_region()->    file: kvm_main.c, line:931
    __kvm_set_memory_region()         file: kvm_main.c, line:734
```

Guest OS访问任意一块物理地址GPA时，都可以通过KVMSlot记载的关系来得到Qemu的虚拟地址映射即HVA,Qemu中地址空间与VM中地址空间的关系如下图：  
![](http://7kttfc.com1.z0.glb.clouddn.com/4.png)

### 虚拟网卡 {#虚拟网卡}

为什么先要提下Guest OS的物理内存管理呢，因为作为一个硬件设备，OS要控制其必然通过PIO与MMIO来与其交互。而目前的网卡主要涉及到MMIO，而且还是PCI接口，所以必然落入图中的PCI mem。

Qemu根据传入的创建指定NIC类型的参数来进行指定NIC的创建操作，对于e1000虚拟网卡来说，其通过type\_init\(e1000\_register\_types\) 注册了MODULE\_INIT\_QOM类型的设备，而当Qemu创建e1000虚拟设备时，通过内部的PCI Bus设备抽象层来进行了e1000的初始化，该抽象层这里不做过多的描述。主要关注网卡部分的初始化：  


```
static TypeInfo e1000_info = {         file: e1000.c, line: 1289
    .name = "e1000",
    .parent = TYPE_PCI_DEVICE,
    .instance_size = sizeof(E1000State),
    .class_init = e1000_class_init,      <-------初始化入口函数
};

static void e1000_class_init(ObjectClass *klass, void *data)   file: e1000.c, line: 1271
{
    DeviceClass *dc = DEVICE_CLASS(klass);
    PCIDeviceClass *k = PCI_DEVICE_CLASS(klass);
    k->init = pci_e1000_init;            <-------注册虚拟网卡pci层的初始化函数
    k->exit = pci_e1000_uninit;
    k->romfile = "pxe-e1000.rom";
    k->vendor_id = PCI_VENDOR_ID_INTEL;
    k->device_id = E1000_DEVID;
    k->revision = 0x03;
    k->class_id = PCI_CLASS_NETWORK_ETHERNET;
    dc->desc = "Intel Gigabit Ethernet";
    dc->reset = qdev_e1000_reset;
    dc->vmsd = &vmstate_e1000;
    dc->props = e1000_properties;
}

static int pci_e1000_init(PCIDevice *pci_dev)
{
    e1000_mmio_setup(d);        <-------e1000的mmio访问建立
    pci_register_bar(&d->dev, 0, PCI_BASE_ADDRESS_SPACE_MEMORY, &d->mmio);   <-------注册mmio空间
    pci_register_bar(&d->dev, 1, PCI_BASE_ADDRESS_SPACE_IO, &d->io);   <-------注册pio空间
    d->nic = qemu_new_nic(&net_e1000_info, &d->conf, object_get_typename(OBJECT(d)), d->dev.qdev.id, d);   
    <----初始化nic信息，并注册虚拟网卡的相关操作函数，结构如下，同时创建了与虚拟网卡对应的net client结构。在
    add_boot_device_path(d->conf.bootindex, &pci_dev->qdev, "/ethernet-phy@0");   加入系统启动设备配置中
}

static NetClientInfo net_e1000_info = {
    .type = NET_CLIENT_OPTIONS_KIND_NIC,
    .size = sizeof(NICState),
    .can_receive = e1000_can_receive,
    .receive = e1000_receive,       <----------主要是receive函数
    .cleanup = e1000_cleanup,
    .link_status_changed = e1000_set_link_status,
};
```

最后PCI设备抽象层将e1000代表的net client与上文描述的e1000所占用的NICinfo所对应hub port的NetClientState \*peer进行关联。

至此就完成了虚拟网卡的建立,最终在Qemu的hub中的各Nic的关系如下：  
![](http://7kttfc.com1.z0.glb.clouddn.com/5.png)

