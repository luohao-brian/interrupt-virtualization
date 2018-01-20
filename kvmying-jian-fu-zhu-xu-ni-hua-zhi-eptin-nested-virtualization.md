> 在嵌套虚拟环境\(Nested Virtualization\)下，运行在hypervisor上的Virtual Machine仍可以作为hypervisor去运行其它的Virutal Machine，而KVM也支持了这种强大的特性。

而在《KVM硬件辅助虚拟化之 EPT》一文中，我们详细分析了单层虚拟机并引入硬件辅助虚拟化EPT功能的环境下，Guest OS中的虚拟地址到真实物理地址的访问方法，即在EPT页表的帮助下，通过二维的页表机制，最终实现GVA到HPA的转换。那么在多层嵌套虚拟机情况下，EPT又是如何发挥作用的呢？

## 引言 {#引言}

关于Nested Virtualization背后的详细理论细节可参看：  
“The Turtles Project: Design and Implementation of Nested Virtualization”一文。  
附:[论文](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Ben-Yehuda.pdf),[视频](https://www.usenix.org/conference/osdi10/turtles-project-design-and-implementation-nested-virtualization)

## 背景描述 {#背景描述}

### 单级结构 {#单级结构}

![](http://7jpohi.com1.z0.glb.clouddn.com/1.png)

在单级VM的情况下，Guest OS的进入由Host Hypervisor发起VM-Launch指令切换到no root mode,而Guest OS发生异常退出时，返回到Hypervisor中进行处理，在EPT页表缺失情况下，VM-Exit的异常原因为**EXIT\_REASON\_EPT\_VIOLATION**，而在对应处理函数中进行了EPT页表的修补动作。

### 多级结构 {#多级结构}

多级结构下，由于二级结构比较典型，所以这里以二级机构进行分析，多级结构以此类推。

二层的VM到底是如何运行的呢？在“The Turtles Project: Design and Implementation of Nested Virtualization”一文中讲述了基本的原理，作为nested EPT的基础，这里简单介绍下。

![](http://7jpohi.com1.z0.glb.clouddn.com/2.png)

如图中所示:

* L0:Host Hypervisor运行级
* L1:Guest Hypervisor运行级
* L2:Guest OS运行级

L0为运行L1，构建VMCS1-&gt;0结构,通过VM-Launch发起运行,图中步骤\(1\)；而L1为运行L2，也需构建VMCS1-2结构，同样通过VM-Launch发起运行,图中步骤\(2\)。然而在no root mode下，VM-Launch为特权级指令，L1会发生VM-Exit,图中步骤\(3\)，其退出原因为EXIT\_REASON\_VMLAUNCH。

```
static int (*const kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = {
    ......
    EXIT_REASON_VMLAUNCH]  = handle_vmlaunch,
    ......
}
static int handle_vmlaunch(struct kvm_vcpu *vcpu)
{
    return nested_vmx_run(vcpu, true);
}
static int nested_vmx_run(struct kvm_vcpu *vcpu, bool launch)
{
    ......
    vmcs12 = get_vmcs12(vcpu);               //VMM L0获取了L1中构建的VMCS1->2
    ......                                   //do some check
    vmcs02 = nested_get_current_vmcs02(vmx); //VMM L0创建了VMCS0->2
    ......
    prepare_vmcs02(vcpu, vmcs12);            //通过VMCS1->2中的信息在L0中构建了VMCS0->2所需的信息
}
```

当上述函数返回后，L0中的流程就是使用VMCS0-&gt;2发起了VM-Launch,图中步骤\(4\)，切换到了GUEST OS，即L2中；  
对于L2看来，就好像是L1是其Hypervisor。

## 嵌套EPT流程分析 {#嵌套EPT流程分析}

### 各级EPT的分配 {#各级EPT的分配}

![](http://7jpohi.com1.z0.glb.clouddn.com/3.png)

L0为支持L1的内存访问，构建了EPT页表，这里记作**EPT0-&gt;1**\(步骤1\);L1若发生EPT页表缺失，会返回到Host Hyprevisor中进行EPT的修补。  
而L1为支持L2的内存访问，也构建了EPT页表，这里记作**EPT1-&gt;2**\(步骤2\);正如上面分析的一样,L1的VM-Launch会被Host Hyperviser接管\(步骤3\)，由L0进行VM-Launch，此时L0会为L2分配**EPT0-&gt;2**\(步骤4，分配的地方看参加前一篇EPT的分析\)，所以真正被L2载入的EPT页表是L0分配的，而从L1角度看上去用的是L1分配的。

### 相关的初始化 {#相关的初始化}

在L0准备VMCS0-&gt;2的时候会准备新的EPT0-&gt;2,注册新的EPT miss的相关处理函数。

```
static void prepare_vmcs02(struct kvm_vcpu *vcpu, struct vmcs12 *vmcs12)
{
    ......
    if (nested_cpu_has_ept(vmcs12)) {
        kvm_mmu_unload(vcpu);
        nested_ept_init_mmu_context(vcpu);
    }
    ......
    kvm_set_cr3(vcpu, vmcs12->guest_cr3);
    kvm_mmu_reset_context(vcpu);
    ......
}
static void nested_ept_init_mmu_context(struct kvm_vcpu *vcpu)
{
    kvm_init_shadow_ept_mmu(vcpu, &vcpu->arch.mmu, nested_vmx_ept_caps & VMX_EPT_EXECUTE_ONLY_BIT);

    vcpu->arch.mmu.set_cr3           = vmx_set_cr3;          
    vcpu->arch.mmu.get_cr3           = nested_ept_get_cr3;
    vcpu->arch.mmu.inject_page_fault = nested_ept_inject_page_fault;

    vcpu->arch.walk_mmu              = &vcpu->arch.nested_mmu;
}    
void kvm_init_shadow_ept_mmu(struct kvm_vcpu *vcpu, struct kvm_mmu *context, bool execonly)
{
    ......
    context->shadow_root_level = kvm_x86_ops->get_tdp_level();
    context->nx = true;
    context->page_fault = ept_page_fault;
    context->gva_to_gpa = ept_gva_to_gpa;
    context->sync_page = ept_sync_page;
    context->invlpg = ept_invlpg;
    context->update_pte = ept_update_pte;
    context->root_level = context->shadow_root_level;
    context->root_hpa = INVALID_PAGE;
    context->direct_map = false;
    ......
}
```

上述EPT打头的函数，比如ept\_page\_fault\(\),ept\_gva\_to\_gpa\(\)均在文件paging\_tmpl.h中.

### EPT miss的处理 {#EPT_miss的处理}

![](http://7jpohi.com1.z0.glb.clouddn.com/4.png)

我们从L2发起内存访问说起，当L2中需要对L2-GVA进行访问：

* 首先，L2会查找L2 OS管理的进程页表，若L2自身的页表发生缺页时，L2不会产生VM-Exit，而是由L2 OS自己进行页表的修补，并产生了对应L2-GVA对应的L2-GPA;

* MMU会使用L2-GPA查找L2VM对应的EPT页表进而得到对应的HPA，而此时该对应的EPT页表是L0为其准备的EPT0-&gt;2;

* 若EPT0-&gt;2中L2-GPA相关的页表项或页表缺失，则会产生EPT Violation原因的VM-Exit；该VM-Exit会直接退出到L0中，见上图步骤1；

* 对应于EXIT\_REASON\_EPT\_VIOLATION，依旧是熟悉的handle\_ept\_violation；

我们进入handle\_ept\_violation进行分析：

```
static int handle_ept_violation(struct kvm_vcpu *vcpu)
{
    ......
    gpa = vmcs_read64(GUEST_PHYSICAL_ADDRESS);
    return kvm_mmu_page_fault(vcpu, gpa, error_code, NULL, 0);
}
int kvm_mmu_page_fault(struct kvm_vcpu *vcpu, gva_t cr2, u32 error_code, void *insn, int insn_len)
{
    ......
    r = vcpu->arch.mmu.page_fault(vcpu, cr2, error_code, false);
    ......
}
```

vcpu-&gt;arch.mmu.page\_fault\(\)即为初始化过程中注册的ept\_page\_fault\(\)，直接到paging\_tmpl.h中找到该函数，该函数是由宏拼接而成：

```
static int FNAME(page_fault)(struct kvm_vcpu *vcpu, gva_t addr, u32 error_code, bool prefault)
{
    //Look up the guest pte for the faulting address.
    r = FNAME(walk_addr)(&walker, vcpu, addr, error_code); //addr为L2-GPA
    ......
    if (!r) {
        inject_page_fault(vcpu, &walker.fault);
    }
    return 0;
}
}
static int FNAME(walk_addr)(struct guest_walker *walker, struct kvm_vcpu *vcpu, gva_t addr, u32 access)
{
    return FNAME(walk_addr_generic)(walker, vcpu, &vcpu->arch.mmu, addr, access);
}
```

我们重点看下ept\_walk\_addr\_generic\(\)函数：

![](http://7jpohi.com1.z0.glb.clouddn.com/5.PNG)

从上述流程可以看出ept\_walk\_addr\_generic\(\)的主要功能就是遍历L1中为L2设置的EPT1-&gt;2,而EPT1-&gt;2页表地址均是L1的GPA,则L0将其转换为L0的HVA，从而访问其各级页表项，若有一级不存在，则退出；  
ept\_walk\_addr返回0，在ept\_page\_fault\(\)进入inject\_page\_fault\(\) —&gt; nested\_ept\_inject\_page\_fault\(\);

```
satic void nested_ept_inject_page_fault(struct kvm_vcpu *vcpu, struct x86_exception *fault)
{
    struct vmcs12 *vmcs12 = get_vmcs12(vcpu);
    u32 exit_reason;
    exit_reason = EXIT_REASON_EPT_VIOLATION;
    //填充相关信息，更改VMCS12，准备运行L1，为的是让L1认为L2是其真正的Guest
    nested_vmx_vmexit(vcpu, exit_reason, 0, vcpu->arch.exit_qualification);
    vmcs12->guest_physical_address = fault->address;
}
```

nested\_ept\_inject\_page\_fault\(\)将设置VM-Exit退出原因EXIT\_REASON\_EPT\_VIOLATION，设置访问出错地址，见上图步骤2；这样就完成了L1中的EPT Violation的异常注入，等到L1开始运行时，就会发生EPT Violation的异常。这样从L1角度看来，就好像是L2发生了EPT Violation的退出。而在L1中的修复EPT1-&gt;2的流程可参考前文描述。

当L2再恢复运行时，又会再次访问L2-GVA,同样会陷入之前一样的流程中，见上图步骤3，而在L1中的ept\_walk\_addr\_generic\(\)函数中由于前一步L1已经修复了其为L1准备的EPT1-&gt;2中各级页表项，则此时ept\_walk\_addr会获得完整的表项信息，  
因此在退出上述大循环的时候，walker里面会携带好一条地址访问在每级页表中的表项内容和相关信息，见上图步骤4.

```
static int FNAME(page_fault)(struct kvm_vcpu *vcpu, gva_t addr, u32 error_code, bool prefault)
{    
    ......
    r = FNAME(walk_addr)(&walker, vcpu, addr, error_code);
    if (!r) {  //此时r为1，则不会发生page_fault异常的注入
        inject_page_fault(vcpu, &walker.fault);
        return 0;
    }

    //至此，walker结构中携带了一条地址访问在每级页表中的完整信息

    try_async_pf(vcpu, prefault, walker.gfn, addr, &pfn, write_fault, &map_writable);
    ......
    FNAME(fetch)(vcpu, addr, &walker, write_fault, level, pfn, map_writable, prefault);
    ......

    return 0
}
```

try\_async\_pf\(\)分配L0的真实物理页面来完成L1 EPT1-&gt;2所使用的L1-GPA到L0-HPA的映射，而ept\_fetch\(\)通过mmu\_set\_spte\(\)会一步一步的讲L1中每级的GPA对应的HPA，来填入L0的EPT页表，完成EPT0-&gt;2缺失的修补。

至此，当L2再次发生GVA的访问时，EPT0-&gt;2中已有对应GPA的HPA表项映射关系，则可以进行正常的地址访问，见上图步骤5。

### Future Discussion {#Future_Discussion}

三级VM的情况下，VM的运行和EPT机制的运作是如何的呢？

![](http://7jpohi.com1.z0.glb.clouddn.com/6.png)

