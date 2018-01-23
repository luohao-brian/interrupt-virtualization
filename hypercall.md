> 在Linux中，大家应该对syscall非常的了解和熟悉，其是用户态进入内核态的一种途径或者说是一种方式，完成了两个模式之间的切换；而在虚拟环境中，有没有一种类似于syscall这种方式，能够从no root模式切换到root模式呢？答案是肯定的，KVM提供了Hypercall机制，x86体系架构也有相关的指令支持。



## Hypercall的发起 {#Hypercall的发起}

KVM代码中提供了五种形式的Hypercall接口：

```
file: arch/x86/include/asm/kvm_para.h, line: 34
static inline long kvm_hypercall0(unsigned int nr);
static inline long kvm_hypercall1(unsigned int nr, unsigned long p1);
static inline long kvm_hypercall2(unsigned int nr, unsigned long p1, unsigned long p2);
static inline long kvm_hypercall3(unsigned int nr, unsigned long p1, unsigned long p2, unsigned long p3)
static inline long kvm_hypercall4(unsigned int nr, unsigned long p1, unsigned long p2, unsigned long p3, unsigned long p4)
```

这几个接口的区别在于参数个数的不用，本质是一样的。挑个参数最多的看下：

```
static inline long kvm_hypercall4(unsigned int nr, unsigned long p1,
				  unsigned long p2, unsigned long p3,
				  unsigned long p4)
{
	long ret;
	asm volatile(KVM_HYPERCALL
		     : "=a"(ret)
		     : "a"(nr), "b"(p1), "c"(p2), "d"(p3), "S"(p4)
		     : "memory");
	return ret;
}
```

Hypercall内部实现是标准的内嵌汇编,稍作分析：

### KVM\_HYPERCALL {#KVM_HYPERCALL}

```
#define KVM_HYPERCALL ".byte 0x0f,0x01,0xc1"
```

对于KVM hypercall来说，KVM\_HYPERCALL是一个三字节的指令序列，x86体系架构下即是vmcall指令，官方手册解释：

```
vmcall：
    op code:0F 01 C1 -- VMCALL Call to VM
 monitor 
by causing VM exit
```

言简意赅，vmcall会导致VM exit到VMM。

### 返回值 {#返回值}

: “=a”\(ret\)，表示返回值放在eax寄存器中输出。

### 输入 {#输入}

: “a”\(nr\), “b”\(p1\), “c”\(p2\), “d”\(p3\), “S”\(p4\),表示输入参数放在对应的eax，ebx，ecx，edx，esi中，而nr其实就是可以认为是系统调用号。

## hypercall的处理 {#hypercall的处理}

当Guest发起一次hypercall后，VMM会接管到该call导致的VM Exit。

```
static int (*const kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = {
	......
	[EXIT_REASON_VMCALL]                  = handle_vmcall,
	......
}
```

进入handle\_vmcall\(\)-&gt;kvm\_emulate\_hypercall\(\)处理，过程非常简单：

```
int kvm_emulate_hypercall(struct kvm_vcpu *vcpu)
{
	unsigned long nr, a0, a1, a2, a3, ret;
	int r = 1;

	if (kvm_hv_hypercall_enabled(vcpu->kvm))
		return kvm_hv_hypercall(vcpu);

	nr = kvm_register_read(vcpu, VCPU_REGS_RAX);  //取出参数
	a0 = kvm_register_read(vcpu, VCPU_REGS_RBX);
	a1 = kvm_register_read(vcpu, VCPU_REGS_RCX);
	a2 = kvm_register_read(vcpu, VCPU_REGS_RDX);
	a3 = kvm_register_read(vcpu, VCPU_REGS_RSI);

	trace_kvm_hypercall(nr, a0, a1, a2, a3);

	if (!is_long_mode(vcpu)) {
		nr &= 0xFFFFFFFF;
		a0 &= 0xFFFFFFFF;
		a1 &= 0xFFFFFFFF;
		a2 &= 0xFFFFFFFF;
		a3 &= 0xFFFFFFFF;
	}

	if (kvm_x86_ops->get_cpl(vcpu) != 0) {
		ret = -KVM_EPERM;
		goto out;
	}

	switch (nr) {                 //根据调用号进行处理
	case KVM_HC_VAPIC_POLL_IRQ:
		ret = 0;
		break;
	case KVM_HC_KICK_CPU:
		kvm_pv_kick_cpu_op(vcpu->kvm, a0, a1);
		ret = 0;
		break;
	default:
		ret = -KVM_ENOSYS;
		break;
	}
out:
	kvm_register_write(vcpu, VCPU_REGS_RAX, ret);   //将处理结果写入eax寄存器返回
	++vcpu->stat.hypercalls;
	return r;
}
```

### Conclusion {#Conclusion}

整个过程非常简洁和简单，hypercall机制给了Guest能够主动进入VMM的一种方式。个人理解是：一旦Guest使用了这种机制，意味着Guest知道自己位于Guest中，也就意味着所谓的半虚拟化。

