> 传统OS环境中，CPU对内存的访问都必须通过MMU将虚拟地址VA转换为物理地址PA从而得到真正的Physical Memory Access,即：**VA-&gt;MMU-&gt;PA**，见下图。

虚拟运行环境中由于Guest OS所使用的物理地址空间并不是真正的物理内存，而是由VMM供其所使用一层虚拟的物理地址空间，为使MMU能够正确的转换虚实地址，Guest中的地址空间的转换和访问都必须借助VMM来实现，这就是内存虚拟化的主要任务，即：**GVA-&gt;MMU Virtualation-&gt;HPA**，见下图。



![](http://7jpnjo.com1.z0.glb.clouddn.com/11.PNG)

## MMU虚拟化方案 {#MMU虚拟化方案}

内存虚拟化，也可以称为MMU的虚拟化，目前有两种方案：

* 影子页表\(Shadow Page Table\)

影子页表是纯软件的MMU虚拟化方案，Guest OS维护的页表负责GVA到GPA的转换，而KVM会维护另外一套影子页表负责GVA到HPA的转换。真正被加载到物理MMU中的页表是影子页表。  
在多进程Guest OS中，每个进程有一套页表，进程切换时也需要切换页表，这个时候就需要清空整个TLB，使所有影子页表的内容无效。但是某些影子页表的内容可能很快就会被再次用到，而重建影子页表是一项十分耗时的工作，因此又需要缓存影子页表。  
缺点: 实现复杂，会出现高频率的VM Exit还需要考虑影子页表的同步，缓存影子页表的内存开销大。

* EPT\(Extended Page Table\)

为了解决影子页表的低效，VT-x\(Intel虚拟化技术方案\)提供了Extended Page Table\(EPT\)技术,直接在硬件上支持了GVA-&gt;GPA-&gt;HPA的两次地址转换，大大降低了内存虚拟化的难度，也大大提高了性能。**本文主要讲述EPT支持下的内存虚拟化方案**。

![](http://7jpnjo.com1.z0.glb.clouddn.com/EPT9.png)

## EPT原理 {#EPT原理}

### 地址空间说明 {#地址空间说明}

* GVA: Guest Virtual Address
* GPA: Guest Physical Address
* HVA: Host Virtual Address
* HPA: Host Physical Address

### 原理描述 {#原理描述}

这里假设Guest OS页表和EPT页表都是4级页表，CPU完成一次地址转换的基本过程如下：

![](http://7jpnjo.com1.z0.glb.clouddn.com/EPT1.png)

如上图所示：

1. CPU首先查找Guest CR3指向的L4页表；
2. 由于Guest CR3给出的是GPA，CPU需要查EPT页表；
3. 如果EPT页表中不存在该地址对应的查找项，则Guest Mode产生EPT Violation异常由VMM来处理；
4. 获取L4页表地址后，CPU根据GVA和L4页表项的内容，来获取L3页表项的GPA;
5. 如果L4页表中GVA对应的表项显示为缺页，那么CPU产生Page Fault,直接交由Guest Kernel处理，注意这里不会产生VM-Exit；
6. 获得L3页表项的GPA后，CPU同样查询EPT页表，过程和上面一样；
7. L2，L1页表的访问也是如此，直至找到最终于与GPA对应的HPA。

## 核心代码说明 {#核心代码说明}

### 初始化 {#初始化}

在内核KVM模块提供的创建VCPU API函数中，进行了虚拟MMU的创建和初始化，调用流程如下所示：

![](http://7jpnjo.com1.z0.glb.clouddn.com/EPT2.PNG)

由于EPT页表操作和影子页表的操作很多步骤上是一致的，所以EPT处理过程中复用了大量的影子页表操作的函数。  
其中比较重要的初始化函数为

```
static void init_kvm_mmu(struct kvm_vcpu *vcpu)
```

当全局变量tdp\_enabled为1时，就会使能EPT功能。

```
static void init_kvm_tdp_mmu(struct kvm_vcpu *vcpu)
{
    struct kvm_mmu *context = vcpu->arch.walk_mmu;

    context->base_role.word = 0;
    context->page_fault = tdp_page_fault;      //EPT Voilation Handle的入口函数
    context->sync_page = nonpaging_sync_page;
    context->invlpg = nonpaging_invlpg;
    context->update_pte = nonpaging_update_pte;
    context->shadow_root_level = kvm_x86_ops->get_tdp_level(); //EPT页表的级数，系统默认为4
    context->root_hpa = INVALID_PAGE;          //EPT页表的指针，初始时为INVALID
    context->direct_map = true;                //直接映射使能
    context->set_cr3 = kvm_x86_ops->set_tdp_cr3;    //很重要，设置CR3，包括GUEST CR3及EPTP的初始化
    context->get_cr3 = get_cr3;
    context->get_pdptr = kvm_pdptr_read;
    context->inject_page_fault = kvm_inject_page_fault;
}
```

### MMU Setup {#MMU_Setup}

在进入Guest之前，KVM会将EPTP与Guest CR3设置好，这样Guest OS才会顺利运行，函数调用关系如下：

![](http://7jpnjo.com1.z0.glb.clouddn.com/EPT3.png)

关键函数：

```
static struct kvm_mmu_page *kvm_mmu_alloc_page(struct kvm_vcpu *vcpu, u64 *parent_pte, int direct)
{
    sp = mmu_memory_cache_alloc(&vcpu->arch.mmu_page_header_cache); //申请struct kvm_mmu_page空间，
                                                                      该结构表示一个EPT页表项
    sp->spt = mmu_memory_cache_alloc(&vcpu->arch.mmu_page_cache);   //申请EPT页表项指向的页空间
    list_add(&sp->link, &vcpu->kvm->arch.active_mmu_pages);         //加入到active_mmu_pages链中
    return sp;
}
static void vmx_set_cr3(struct kvm_vcpu *vcpu, unsigned long cr3)  参数cr3即是vcpu->arch.mmu.root_hpa
{
    unsigned long guest_cr3;
    u64 eptp;
    guest_cr3 = cr3;
    if (enable_ept) {
        eptp = construct_eptp(cr3);      
        vmcs_write64(EPT_POINTER, eptp);       //构造所需的EPTP写入VMCS中，指向EPT页表
        if (is_paging(vcpu) || is_guest_mode(vcpu))      //设置Guest CR3
            guest_cr3 = kvm_read_cr3(vcpu);
        else
            guest_cr3 = vcpu->kvm->arch.ept_identity_map_addr;
        ept_load_pdptrs(vcpu);
    }
    vmx_flush_tlb(vcpu);                      //刷新TLB
    vmcs_writel(GUEST_CR3, guest_cr3);        // 指定CR3，指向Guest的页表
}

```

其中我们也可以看到，如果enable\_ept开关关闭的话，传入的root\_hpa也就直接当Guest CR3用，其实就是影子页表的基址。

OK，上面一切都设置好后，就进入了Guest OS中运行了。

### EPT Violation Handle {#EPT_Violation_Handle}

当CPU访问EPT页表查找HPA时，发现相应的页表项不存在，则会发生EPT Violation异常，导致VM-Exit，返回到VMM中处理。

```
static int (*const kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = 
{
    ......
    [EXIT_REASON_EPT_VIOLATION]          = handle_ept_violation,
    ......
}
```

根据EXIT\_REASON\_EPT\_VIOLATION注册的处理函数handle\_ept\_violation\(\)从VMCS结构中读出GPA进行处理；

```
static int handle_ept_violation(struct kvm_vcpu *vcpu)
{
    ......
    gpa = vmcs_read64(GUEST_PHYSICAL_ADDRESS);
    ......
    return kvm_mmu_page_fault(vcpu, gpa, error_code, NULL, 0);
}

int kvm_mmu_page_fault(struct kvm_vcpu *vcpu, gva_t cr2, u32 error_code, void *insn, int insn_len)
{
    ......
    r = vcpu->arch.mmu.page_fault(vcpu, cr2, error_code, false);
    ......
}
```

此处page\_fault\(\)即初始化注册的tdp\_page\_fault\(\);

```
static int tdp_page_fault(struct kvm_vcpu *vcpu, gva_t gpa, u32 error_code, bool prefault)
{
    ......
    level = mapping_level(vcpu, gfn);   //计算请求的level值，一般为1，即页表的最后一级
    ......
    try_async_pf(vcpu, prefault, gfn, gpa, &pfn, write, &map_writable); //获取GPA对应的HPA
    ......
    r = __direct_map(vcpu, gpa, write, map_writable, level, gfn, pfn, prefault);
    ......
}
```

![](http://7jpnjo.com1.z0.glb.clouddn.com/EPT5.png)

![](http://7jpnjo.com1.z0.glb.clouddn.com/EPT6.png)

for\_each\_shadow\_entry\(\){}不断的遍历页表的层级，如果下一级的页表项不存在，则分配一个EPT页表页，设置缺失的页表项，再进行下一级的页表项的查找，直至达到需要填充的页表级，一般为最后一级。  
将在前面流程中获取的HPA通过mmu\_set\_spte\(\)进行设置。如果原来存在，说明该页表项已经失效，需要更新、覆盖，最后刷新TLB。

再介绍下for\_each\_shadow\_entry\(\){}的流程：

```
struct kvm_shadow_walk_iterator {
    u64 addr;            //gfn << PAGE_SHIFT   Guest物理页基址
    hpa_t shadow_addr;   //当前VM的EPT页表项的物理页机制
    u64 *sptep;          //指向下一级EPT页表的指针
    int level;           //当前所处的页表级别，逐级递减
    unsigned index;      //页表的索引      *sptep + index = 下下一级EPT页表的指针
};

#define for_each_shadow_entry(_vcpu, _addr, _walker)    \
for (shadow_walk_init(&(_walker), _vcpu, _addr);    \
     shadow_walk_okay(&(_walker));            \
     shadow_walk_next(&(_walker)))
```

shadow walk init/okay/next操作见下图：

![](http://7jpnjo.com1.z0.glb.clouddn.com/EPT8.png)

### Conclusion {#Conclusion}

OK，一个完整的EPT初始化、设置、使用流程到此结束。总结一下几个地址空间的关系：

* GVA到GPA的映射关系由Guest OS的页表来维护;
* GPA到HVA的映射关系由KVM的Memory Slot数组来维护\(将在另外的文章中介绍\)；
* HVA到HPA的映射关系由Host OS的页表来维护\(将在另外的文章中介绍，即上文中GPA对应的HPA的获取\)；
* GPA到HPA的映射关系由EPT页表来维护；



