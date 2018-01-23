前面我们对VFIO框架原理进行了简要的介绍，本文则主要探讨一下VFIO实现中的几个技术关键点，主要包括：

1. VFIO中如实现对直通设备的PCI 配置空间，PCI BAR空间，I/O Port，MMIO的访问支持？
2. VFIO中如何实现MSI/MSI-X，Interrupt Remapping，以及Posted Interrupt的支持？
3. VFIO中是如何实现DMA Remapping的？
4. VFIO中是如何支持设备热插拔的？

---

### 1. VFIO中如实现对直通设备的PCI 配置空间，PCI BAR空间，I/O Port，MMIO的访问支持？

值得注意的是直通设备的配置空间并不是直接呈现给虚拟机的，VFIO中会对设备的PCI 配置空间进行模拟。 为什么VFIO不直接把直通PCI 配置空间呈现给虚拟机呢？主要原因是一部分PCI Capability不能直接对guest呈现，VFIO需要截获部分guest驱动对某些PCI配置空间的操作， 另外像MSI/MSIx等特性需要QEMU/KVM的特殊处理，所以也不能直接呈现给虚拟机。

VFIO内核模块和QEMU会相互配合来完成设备的PCI配置空间的模拟，在VFIO中`vfio_config_init`函数中会为PCI设备初始化模拟的PCI配置空间。 对于每个设备而言，VFIO内核模块为其分配了一个`pci_config_map`结构，每个PCI Capability都有一个与之对应的`perm_bits`， 我们可以重写其hook函数来截获对这个Capability的访问（读/写）。

```
int vfio_config_init(struct vfio_pci_device *vdev)
{
    ......
    map = kmalloc(pdev->cfg_size, GFP_KERNEL);
    if (!map)
        return -ENOMEM;

    vconfig = kmalloc(pdev->cfg_size, GFP_KERNEL);
    if (!vconfig) {
        kfree(map);
        return -ENOMEM;
    }

    vdev->pci_config_map = map;
    vdev->vconfig = vconfig;

    memset(map, PCI_CAP_ID_BASIC, PCI_STD_HEADER_SIZEOF);
    memset(map + PCI_STD_HEADER_SIZEOF, PCI_CAP_ID_INVALID,
           pdev->cfg_size - PCI_STD_HEADER_SIZEOF);
    ......
        map = kmalloc(pdev->cfg_size, GFP_KERNEL);
    if (!map)
        return -ENOMEM;

    vconfig = kmalloc(pdev->cfg_size, GFP_KERNEL);
    if (!vconfig) {
        kfree(map);
        return -ENOMEM;
    }   

    vdev->pci_config_map = map;
    vdev->vconfig = vconfig;

    memset(map, PCI_CAP_ID_BASIC, PCI_STD_HEADER_SIZEOF);
    memset(map + PCI_STD_HEADER_SIZEOF, PCI_CAP_ID_INVALID,
           pdev->cfg_size - PCI_STD_HEADER_SIZEOF);

    ret = vfio_cap_init(vdev);   /* 初始化PCI Capability，主要用来填充模拟的vconfig */                                                                                                                                                     
    if (ret)                                                                                                                                                                        
        goto out;                                                                                                                                                                   

    ret = vfio_ecap_init(vdev);  /* 初始化PCI Extended Capability，主要用来填充模拟的vconfig */          
    ......
```

`vfio_pci_init_perm_bits`函数中就已经对不同的PCI Capability访问权限做了设置，例如Power Management这类Capability就做了特殊的权限控制。

```
int __init vfio_pci_init_perm_bits(void)
{   
    int ret;

    /* Basic config space */
    ret = init_pci_cap_basic_perm(&cap_perms[PCI_CAP_ID_BASIC]);

    /* Capabilities */
    ret |= init_pci_cap_pm_perm(&cap_perms[PCI_CAP_ID_PM]);         /* Power Management */
    ret |= init_pci_cap_vpd_perm(&cap_perms[PCI_CAP_ID_VPD]);       /* Vital Product Data */
    ret |= init_pci_cap_pcix_perm(&cap_perms[PCI_CAP_ID_PCIX]);     /* PCIX */
    cap_perms[PCI_CAP_ID_VNDR].writefn = vfio_raw_config_write;     
    ret |= init_pci_cap_exp_perm(&cap_perms[PCI_CAP_ID_EXP]);       /* PCI Express */
    ret |= init_pci_cap_af_perm(&cap_perms[PCI_CAP_ID_AF]);         /* AF FLR: Advanced Feature Capability */

    /* Extended capabilities */
    ret |= init_pci_ext_cap_err_perm(&ecap_perms[PCI_EXT_CAP_ID_ERR]);
    ret |= init_pci_ext_cap_pwr_perm(&ecap_perms[PCI_EXT_CAP_ID_PWR]);
    ecap_perms[PCI_EXT_CAP_ID_VNDR].writefn = vfio_raw_config_write;

    if (ret)
        vfio_pci_uninit_perm_bits();

    return ret;
}
```

同样PCI设备的BAR空间也不是直接呈现给guest，这是为什么呢？我们知道，PCI设备的I/O地址空间是通过PCI BAR空间报告给操作系统的。那么有两种方式来：

（1）将设备的PCI BAR空间直接报告给guest，并通过VMCS的I/O bitmap和EPT使guest访问PCI设备的PIO和MMIO都不需要VM-Exit，这样guest操作系统的驱动程序就可以直接访问设备的I/O地址空间。

（2）建立转换表（重映射）来报告PCI BAR空间给guest，当guest访问PIO或者MMIO时候VMM负责截获并转发I/O请求到设备的I/O地址空间。

上面两种方法中，方法（1）是高效和简单的，但实际上却无法运用，原因很简单： 设备的PCI BAR空间在host上由host BIOS分配的，当这个设备呈现给虚拟机后guest BIOS又会对其PCI BAR空间进行分配，这样一来两者之间就产生了冲突。 在操作系统看来，这样就发生了资源冲突，很可能停用这个设备，甚至还会造成更加严重的后果。 实际上，由于操作系统修改设备PCI BAR空间的权利，我们应该阻止guest来修改真实设备的PCI BAR地址以防止造成host上PCI设备的BAR空间冲突。

我们知道PCI直通设备的PIO和MMIO访问方式是通过BAR空间给guest操作系统报告的，例如，下面这张PCI网卡 BAR0的起始地址为0xef100000，大小为256k，non-prefetchable表明一般为设备Register区域，BAR2的地址为 e000，大小为128字节，为设备的PIO区域。

```
04:00.0 Ethernet controller: Qualcomm Atheros Killer E2500 Gigabit Ethernet Controller (rev 10)
Subsystem: Gigabyte Technology Co., Ltd Device e000
Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx+
Latency: 0, Cache Line Size: 64 bytes
Interrupt: pin A routed to IRQ 17
Region 0: Memory at ef100000 (64-bit, non-prefetchable) [size=256K]
Region 2: I/O ports at e000 [size=128]
```

这里从代码上对VFIO的BAR空间映射做一点分析。QEMU在初始化过程中调用`vfio_get_region_info`来查询设备在host上呈现的BAR空间信息：

```
int vfio_get_region_info(VFIODevice *vbasedev, int index,      
                        struct vfio_region_info **info)        
{
    size_t argsz = sizeof(struct vfio_region_info);

    *info = g_malloc0(argsz);                 

    (*info)->index = index;
retry:                                                       
    (*info)->argsz = argsz;

    if (ioctl(vbasedev->fd, VFIO_DEVICE_GET_REGION_INFO, *info)) {
        g_free(*info);
        *info = NULL;
        return -errno;
    }                                   

    if ((*info)->argsz > argsz) {
        argsz = (*info)->argsz;
        *info = g_realloc(*info, argsz);

        goto retry;
    }     

    return 0;                           
}
```

查询是通过调用设备fd上的ioctl传入VFIO\_DEVICE\_GET\_REGION\_INFO命令，通过这个调用可以查询到对应BAR空间的size和offset信息。

```
static long vfio_pci_ioctl(void *device_data,
            unsigned int cmd, unsigned long arg)
{
    ......
    } else if (cmd == VFIO_DEVICE_GET_REGION_INFO) {                                                                                                                                
    struct pci_dev *pdev = vdev->pdev;                                                                                                                                          
    struct vfio_region_info info; 
    struct vfio_info_cap caps = { .buf = NULL, .size = 0 };                                                                                                                     
    int i, ret;                                                                                                                                                                 

    minsz = offsetofend(struct vfio_region_info, offset);                                                                                                                       

    if (copy_from_user(&info, (void __user *)arg, minsz))                                                                                                                       
        return -EFAULT;                                                                                                                                                         

    if (info.argsz < minsz)                                                                                                                                                     
        return -EINVAL;                                                                                                                                                         

    switch (info.index) {
    case VFIO_PCI_CONFIG_REGION_INDEX:   /* PCI 配置空间大小 */
        info.offset = VFIO_PCI_INDEX_TO_OFFSET(info.index);                                                                                                                     
        info.size = pdev->cfg_size;
        info.flags = VFIO_REGION_INFO_FLAG_READ |                                                                                                                               
                    VFIO_REGION_INFO_FLAG_WRITE;                                                                                                                                   
        break;
    case VFIO_PCI_BAR0_REGION_INDEX ... VFIO_PCI_BAR5_REGION_INDEX:                                                                                                             
        info.offset = VFIO_PCI_INDEX_TO_OFFSET(info.index);                                                                                                                     
        info.size = pci_resource_len(pdev, info.index);
        if (!info.size) {
            info.flags = 0;
            break;
        }

        info.flags = VFIO_REGION_INFO_FLAG_READ |
                    VFIO_REGION_INFO_FLAG_WRITE;
        if (vdev->bar_mmap_supported[info.index]) {
            info.flags |= VFIO_REGION_INFO_FLAG_MMAP;
            if (info.index == vdev->msix_bar) {
                ret = msix_sparse_mmap_cap(vdev, &caps);
                if (ret)
                    return ret;
            }
        }
        break;
    ......
}
```

QEMU在查询到物理设备的PCI BAR空间信息后，会通过`vfio_region_mmap`对BAR空间进行映射，这样KVM就会为MMIO建立EPT页表，虚拟机后续访问BAR空间就可以不用发生VM-Exit 如此一来就变得很高效。同时还会调用`pci_register_bar`来注册BAR空间信息，type这里可以PIO或者MMIO。

```
static void vfio_bar_setup(VFIOPCIDevice *vdev, int nr)                                                                                                                             
{                                                                                                                                                                                   
    VFIOBAR *bar = &vdev->bars[nr];                                                                                                                                                 

    uint32_t pci_bar;                                                                                                                                                               
    uint8_t type;                                                                                                                                                                   
    int ret;                                                                                                                                                                        

    /* Skip both unimplemented BARs and the upper half of 64bit BARS. */                                                                                                            
    if (!bar->region.size) {                                                                                                                                                        
        return;                                                                                                                                                                     
    }                                                                                                                                                                               

    /* Determine what type of BAR this is for registration */                                                                                                                       
    ret = pread(vdev->vbasedev.fd, &pci_bar, sizeof(pci_bar),                                                                                                                       
                vdev->config_offset + PCI_BASE_ADDRESS_0 + (4 * nr));                                                                                                               
    if (ret != sizeof(pci_bar)) {                                                                                                                                                   
        error_report("vfio: Failed to read BAR %d (%m)", nr);                                                                                                                       
        return;                                                                                                                                                                     
    }                                                                                                                                                                               

    pci_bar = le32_to_cpu(pci_bar);                                                                                                                                                 
    bar->ioport = (pci_bar & PCI_BASE_ADDRESS_SPACE_IO);                                                                                                                            
    bar->mem64 = bar->ioport ? 0 : (pci_bar & PCI_BASE_ADDRESS_MEM_TYPE_64);                                                                                                        
    type = pci_bar & (bar->ioport ? ~PCI_BASE_ADDRESS_IO_MASK :                                                                                                                     
                                    ~PCI_BASE_ADDRESS_MEM_MASK);                                                                                                                    

    if (vfio_region_mmap(&bar->region)) {                                                                                                                                           
        error_report("Failed to mmap %s BAR %d. Performance may be slow",                                                                                                           
                     vdev->vbasedev.name, nr);                                                                                                                                      
    }                                                                                                                                                                               

    pci_register_bar(&vdev->pdev, nr, type, bar->region.mem);                                                                                                                       
}
```

那么聪明的你一定会想，PCI BAR空间重映射是如何完成的呢？

秘密就在`vfio_region_mmap`函数中，我们摘取最关键部分进行简单分析。 VFIO内核中提供了一个名为`vfio_pci_mmap`的函数，该函数中调用了`remap_pfn_range`，将物理设备的PCI BAR重新映射到了进程的虚拟地址空间。 这样进程中，对该区域进行mmap后，拿到虚拟地址后就可以直接通过读写该段地址来访问物理设备的BAR空间。 在该函数中，将该段重映射虚拟地址空间作为RAM Device分配给guest，在memory\_region\_init\_ram\_device\_ptr函数中会将该段内容标志为RAM， 如此一来KVM会调用kvm\_set\_user\_memory\_region为该段内存区域建立GPA到HPA的EPT映射关系，这样一来Guest内部通过MMIO访问设备BAR空间就不用发生VM-Exit。 当QEMU讲该段虚拟地址空间分配给guest后，seabios初始化时枚举PCI设备的时候，会将该段内存区域映射到guest的物理地址空间，guest驱动再将该段区域映射到 guest内核虚拟地址空间通过访问该段区域来驱动设备。

```
int vfio_region_mmap(VFIORegion *region)
{ 
    ......
    for (i = 0; i < region->nr_mmaps; i++) {
        /* 将物理设备的PCI BAR空间mmap到进程的虚拟地址空间内 
         * region->vbasedev->fd 为设备的fd
         */
        region->mmaps[i].mmap = mmap(NULL, region->mmaps[i].size, prot,
                                     MAP_SHARED, region->vbasedev->fd,
                                     region->fd_offset +
                                     region->mmaps[i].offset);
        ......
        name = g_strdup_printf("%s mmaps[%d]",
                               memory_region_name(region->mem), i);
        memory_region_init_ram_device_ptr(&region->mmaps[i].mem,
                                          memory_region_owner(region->mem),
                                          name, region->mmaps[i].size,
                                          region->mmaps[i].mmap);
        g_free(name);
        memory_region_add_subregion(region->mem, region->mmaps[i].offset,
                                    &region->mmaps[i].mem);
        ......
    }

    return 0;
}
```



