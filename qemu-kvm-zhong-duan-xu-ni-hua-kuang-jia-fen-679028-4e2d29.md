本文重点分析中断的处理流程。

#### \(1\)pic中断处理流程

为了一窥中断处理的具体流程，这里我们选择最简单模拟串口为例进行分析。qemu作为设备模拟器会模拟很多传统的设备，isa-serial就是其中之一。我们看下串口触发中断时候的调用栈：

```
#0  0x00005555557dd543 in kvm_set_irq (s=0x5555568f4440, irq=4, level=1) at /home/fang/code/qemu/accel/kvm/kvm-all.c:991
#1  0x0000555555881c0f in kvm_pic_set_irq (opaque=0x0, irq=4, level=1) at /home/fang/code/qemu/hw/i386/kvm/i8259.c:114
#2  0x00005555559cb0aa in qemu_set_irq (irq=0x5555577c9dc0, level=1) at hw/core/irq.c:45
#3  0x0000555555881fda in kvm_pc_gsi_handler (opaque=0x555556b61970, n=4, level=1) at /home/fang/code/qemu/hw/i386/kvm/ioapic.c:55
#4  0x00005555559cb0aa in qemu_set_irq (irq=0x555556b63660, level=1) at hw/core/irq.c:45
#5  0x00005555559c06e7 in qemu_irq_raise (irq=0x555556b63660) at /home/fang/code/qemu/include/hw/irq.h:16
#6  0x00005555559c09b3 in serial_update_irq (s=0x555557b77770) at hw/char/serial.c:145
#7  0x00005555559c138c in serial_ioport_write (opaque=0x555557b77770, addr=1, val=2, size=1) at hw/char/serial.c:404
```

可以看到qemu用户态有个函数_kvm\_set\_irq_，这个函数是用户态通知kvm内核态触发一个中断的入口。函数中通过调用 kvm\_vm\_ioctl注入一个中断，调用号是 KVM\_IRQ\_LINE（pic类型中断），入参是一个 kvm\_irq\_level 的数据结构（传入了irq编号和中断的电平信息）。模拟isa串口是个isa设备使用边沿触发，所以注入中断会调用2次这个函数前后2次电平相反。

```
int kvm_set_irq(KVMState *s, int irq, int level)
{
    struct kvm_irq_level event;
    int ret;

    assert(kvm_async_interrupts_enabled());

    event.level = level;
    event.irq = irq;
    ret = kvm_vm_ioctl(s, s->irq_set_ioctl, &event);
    if (ret < 0) {
        perror("kvm_set_irq");
        abort();
    }

    return (s->irq_set_ioctl == KVM_IRQ_LINE) ? 1 : event.status;
}
```

这个ioctl在内核的处理对应到下面这段代码，pic类型中断进而会调用到kvm\_vm\_ioctl\_irq\_line函数。

```
kvm_vm_ioctl
{
    ......
#ifdef __KVM_HAVE_IRQ_LINE
    case KVM_IRQ_LINE_STATUS:
    case KVM_IRQ_LINE: {            /* 处理pic上产生的中断 */
        struct kvm_irq_level irq_event;

        r = -EFAULT;
        if (copy_from_user(&irq_event, argp, sizeof(irq_event)))
            goto out;

        r = kvm_vm_ioctl_irq_line(kvm, &irq_event,
                    ioctl == KVM_IRQ_LINE_STATUS);
        if (r)
            goto out;

        r = -EFAULT;
        if (ioctl == KVM_IRQ_LINE_STATUS) {
            if (copy_to_user(argp, &irq_event, sizeof(irq_event)))
                goto out;
        }

        r = 0;
        break;
    }
#endif
#ifdef CONFIG_HAVE_KVM_IRQ_ROUTING      /* 处理ioapic的中断 */
    case KVM_SET_GSI_ROUTING: {
        struct kvm_irq_routing routing;
        struct kvm_irq_routing __user *urouting;
        struct kvm_irq_routing_entry *entries = NULL;

        r = -EFAULT;
        if (copy_from_user(&routing, argp, sizeof(routing)))
            goto out;
        r = -EINVAL;
        if (!kvm_arch_can_set_irq_routing(kvm))
            goto out;
        if (routing.nr > KVM_MAX_IRQ_ROUTES)
            goto out;
        if (routing.flags)
            goto out;
        if (routing.nr) {
            r = -ENOMEM;
            entries = vmalloc(routing.nr * sizeof(*entries));
            if (!entries)
                goto out;
            r = -EFAULT;
            urouting = argp;
            if (copy_from_user(entries, urouting->entries,
                    routing.nr * sizeof(*entries)))
                goto out_free_irq_routing;
        }
        r = kvm_set_irq_routing(kvm, entries, routing.nr,
                    routing.flags);
out_free_irq_routing:
        vfree(entries);
        break;
    }
    ......
}
```

kvm\_vm\_ioctl\_irq\_line函数中会进一步调用内核态的_kvm\_set\_irq_函数（用户态用同名函数额），这个函数是整个中断处理的入口：

```
int kvm_vm_ioctl_irq_line(struct kvm *kvm, struct kvm_irq_level *irq_event,
            bool line_status)
{
    if (!irqchip_in_kernel(kvm))
        return -ENXIO;

    irq_event->status = kvm_set_irq(kvm, KVM_USERSPACE_IRQ_SOURCE_ID,
                    irq_event->irq, irq_event->level,
                    line_status);
    return 0;
}
```

kvm\_set\_irq函数的入参有5个，kvm代表某个特性的的虚拟机，irq\_source\_id可以是KVM\_USERSPACE\_IRQ\_SOURCE\_ID或者KVM\_IRQFD\_RESAMPLE\_IRQ\_SOURCE\_ID\(这个是irqfd这个我们这里不讨论\)，irq是传入的设备irq号，对于串口来说第一个port的irq=4而且irq=gsi，level代表电平。kvm\_irq\_map函数会获取改gsi索引上注册的中断路由项\(kvm\_kernel\_irq\_routing\_entry\)，while循环中会挨个调用每个中断路由项上的set方法触发中,_在guest中会忽略没有实现的芯片类型发送的中断消息_。

对于PIC而言，set函数对应于kvm\_set\_pic\_irq函数，对于IOAPIC而言set函数对应于kvm\_set\_ioapic\_irq（不同的chip不一样额）。对于串口而言，我们会进一步调用kvm\_pic\_set\_irq来处理中断。

```
/*
* Return value:
*  < 0   Interrupt was ignored (masked or not delivered for other reasons)
*  = 0   Interrupt was coalesced (previous irq is still pending)
*  > 0   Number of CPUs interrupt was delivered to
*/
int kvm_set_irq(struct kvm *kvm, int irq_source_id, u32 irq, int level,
        bool line_status)
{
    struct kvm_kernel_irq_routing_entry irq_set[KVM_NR_IRQCHIPS];
    int ret = -1, i, idx;

    trace_kvm_set_irq(irq, level, irq_source_id);

    /* Not possible to detect if the guest uses the PIC or the
    * IOAPIC.  So set the bit in both. The guest will ignore
    * writes to the unused one.
    */
    idx = srcu_read_lock(&kvm->irq_srcu);
    i = kvm_irq_map_gsi(kvm, irq_set, irq);
    srcu_read_unlock(&kvm->irq_srcu, idx);

    /* 依次调用同一个gsi上的所有芯片的set方法 */
    while (i--) {
        int r;
        r = irq_set[i].set(&irq_set[i], kvm, irq_source_id, level,
                line_status);
        if (r < 0)
            continue;

        ret = r + ((ret < 0) ? 0 : ret);
    }

    return ret;
}

/* 查询出此gsi号上对应的所有的“中断路由项” */
int kvm_irq_map_gsi(struct kvm *kvm,
            struct kvm_kernel_irq_routing_entry *entries, int gsi)
{
    struct kvm_irq_routing_table *irq_rt;
    struct kvm_kernel_irq_routing_entry *e;
    int n = 0;

    irq_rt = srcu_dereference_check(kvm->irq_routing, &kvm->irq_srcu,
                    lockdep_is_held(&kvm->irq_lock));
    if (irq_rt && gsi < irq_rt->nr_rt_entries) {
        hlist_for_each_entry(e, &irq_rt->map[gsi], link) {  /* 遍历此gsi对应的中断路由项 */
            entries[n] = *e;
            ++n;
        }
    }

    return n;
}
```

kvm\_pic\_set\_irq 函数中，根据传入的irq编号check下原先的irq\_state将本次的level与上次的irq\_state进行逻辑“异或”判断是否发生电平跳变从而进行边沿检测（pic\_set\_irq1）。如果是的话设置IRR对应的bit，然后调用和pic\_update\_irq更新pic相关的寄存器并唤醒vcpu注入中断。

```
int kvm_pic_set_irq(struct kvm_pic *s, int irq, int irq_source_id, int level)
{
    int ret, irq_level;

    BUG_ON(irq < 0 || irq >= PIC_NUM_PINS);

    pic_lock(s);
    /* irq_level = 1表示该irq引脚有电平跳变，出发中断 */
    irq_level = __kvm_irq_line_state(&s->irq_states[irq],
                    irq_source_id, level);
    /* 一个pic最多8个irq */
    ret = pic_set_irq1(&s->pics[irq >> 3], irq & 7, irq_level);
    pic_update_irq(s);
    trace_kvm_pic_set_irq(irq >> 3, irq & 7, s->pics[irq >> 3].elcr,
                s->pics[irq >> 3].imr, ret == 0);
    pic_unlock(s);

    return ret;
}
```

最后的最后，pic\_unlock函数中在如果wakeup为true（又中断产生时）的时候会遍历每个vcpu，在满足条件的情况下调用_kvm\_make\_request_为vcpu注入中断，然后kick每个vcpu。

```
static void pic_unlock(struct kvm_pic *s)
    __releases(&s->lock)
{
    bool wakeup = s->wakeup_needed;
    struct kvm_vcpu *vcpu;
    int i;

    s->wakeup_needed = false;

    spin_unlock(&s->lock);

    if (wakeup) {       /* wakeup在pic_update_irq中被更新 */
        kvm_for_each_vcpu(i, vcpu, s->kvm) {
            if (kvm_apic_accept_pic_intr(vcpu)) {
                /* 中断注入会在kvm_enter_guest时候执行 */
                kvm_make_request(KVM_REQ_EVENT, vcpu);
                kvm_vcpu_kick(vcpu);
                return;
            }
        }
    }
}
```



