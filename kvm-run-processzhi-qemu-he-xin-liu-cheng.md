> 本文主要介绍在整个创建和运行过程中Qemu与KVM里两者的核心运行流程。

## Qemu核心流程 {#Qemu核心流程}

### 阶段一：参数解析 {#阶段一：参数解析}

这里使用qemu版本为qemu-kvm-1.2.0,使用Qemu工具使能虚拟机运行的命令为：

```
$sudo /usr/local/kvm/bin/qemu-system-x86_64 -hda vdisk_linux.img -m 1024
```

这时候会启动qemu-system-x86\_64应用程序，该程序入口为

```
int main(int argc, char **argv, char **envp)   <------file: vl.c,line: 2345
```

在main函数中第一阶段主要对命令传入的参数进行parser，包括如下几个方面：

```
QEMU_OPTION_M                      机器类型及体系架构相关
QEMU_OPTION_hda
/mtdblock/
pflash    存储介质相关
QEMU_OPTION_numa                   numa系统相关
QEMU_OPTION_kernel                 内核镜像相关
QEMU_OPTION_initrd                 initramdisk相关
QEMU_OPTION_append                 启动参数相关
QEMU_OPTION_net/netdev             网络相关
QEMU_OPTION_smp                    smp相关
```

### 阶段二：VM的创建 {#阶段二：VM的创建}

通过configure\_accelerator\(\)-&gt;kvm\_init\(\) file: kvm-all.c, line: 1281;  
首先打开_/dev/kvm_，获得三大描述符之一kvmfd， 其次通过_KVM\_GET\_API\_VERSION_进行版本验证，最后通过_KVM\_CREATE\_VM_创建了一个VM对象，返回了三大描述符之二: VM描述符/vmfd。

```
s->fd = qemu_open("/dev/kvm", O_RDWR);        kvm_init()/line: 1309
ret = kvm_ioctl(s, KVM_GET_API_VERSION, 0);   kmv_init()/line: 1316
s->vmfd = kvm_ioctl(s, KVM_CREATE_VM, 0);     kvm_init()/line: 1339
```

### 阶段三：VM的初始化 {#阶段三：VM的初始化}

通过加载命令的参数的解析和相关系统的初始化，找到对应的machine类型进行第三阶段的初始化：

```
machine->init(ram_size, boot_devices, kernel_filename, kernel_cmdline, initrd_filename, cpu_model);
file: vl.c, line: 3651
```

其中的参数包括从命令行传入解析后得到的，ram的大小，内核镜像文件名，内核启动参数，initramdisk文件名，cpu模式等。  
我们使用系统默认类型的machine，则init函数为pc\_init\_pci\(\),通过一系列的调用：

```
pc_init_pci()   file: pc_piix.c, line: 294
    --->pc_init1()    file: pc_piix.c, line: 123
        --->pc_cpus_init()  file: pc.c, line: 941
```

在命令行启动时配置的smp参数在这里启作用了，qemu根据配置的cpu个数，进行n次的cpu初始化，相当于n个核的执行体。

```
void pc_cpus_init(const char *cpu_model)
{
    int i;
    /* init CPUs */
for(i = 0; i < smp_cpus; i++) {
        pc_new_cpu(cpu_model);
    }
}
```

继续cpu的初始化：

```
pc_new_cpu()    file: hw/pc.c, line: 915
    --->cpu_x86_init()    file: target-i386/helper.c, line: 1150
        --->x86_cpu_realize()    file: target-i386/cpu.c, line: 1767
            --->qemu_init_vcpu()    file: cpus.c, line: 1039
                --->qemu_kvm_start_vcpu()    file: cpus.c, line: 1011
```

qemu\_kvm\_start\_vcpu是一个比较重要的函数，我们在这里可以看到VM真正的执行体是什么。

### 阶段四：VM RUN {#阶段四：VM_RUN}

```
static void qemu_kvm_start_vcpu(CPUArchState *env) <--------file: cpus.c, line: 1011
{
    CPUState *cpu = ENV_GET_CPU(env);
    ......
    qemu_cond_init(env->halt_cond);
    qemu_thread_create(cpu->thread, qemu_kvm_cpu_thread_fn, env, QEMU_THREAD_JOINABLE);
    ......
}


void qemu_thread_create(QemuThread *thread,
                       void *(*start_routine)(void*),
                       void *arg, int mode)    <--------file: qemu-thread-posix.c, line: 118
{
    ......
    err = pthread_attr_init(&attr);
    ......
	err = pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
	......
    pthread_sigmask(SIG_SETMASK, &set, &oldset);
    ......
    pthread_create(&thread->thread, &attr, start_routine, arg);
    ......
    pthread_attr_destroy(&attr);
}
```

可以看到VM真正的执行体是QEMU进程创建的一系列POSIX线程，而线程执行函数为qemu\_kvm\_cpu\_thread\_fn。  
kvm\_init\_vcpu\(\)通过_KVM\_CREATE\_VCPU_创建了三大描述符之三：vcpu描述符/vcpufd。  
并进入了while\(1\)的循环循环，反复调用kvm\_cpu\_exec\(\)。

```
static void *qemu_kvm_cpu_thread_fn(void *arg)   file: cpus.c, line: 732
{
	......
    r = kvm_init_vcpu(env);       <--------file: kvm-all.c, line: 213
    ......
    qemu_kvm_init_cpu_signals(env);
    /* signal CPU creation */
    env->created = 1;
    qemu_cond_signal(&qemu_cpu_cond);
    while (1) {
        if (cpu_can_run(env)) {
            r = kvm_cpu_exec(env);      <--------file: kvm-all.c, line: 1550 
            if (r == EXCP_DEBUG) {
                cpu_handle_guest_debug(env);
            }
        }
        qemu_kvm_wait_io_event(env);
    }
    return NULL;
}
```

我们可以看到kvm\_cpu\_exec\(\)中又是一个do\(\)while\(ret == 0\)的循环体，该循环体中主要通过_KVM\_RUN_启动VM的运行，从此处进入了kvm的内核处理阶段，并等待返回结果，同时根据返回的原因进行相关的处理，最后将处理结果返回。因为整个执行体在上述函数中也是在循环中，所以后续又会进入到该函数的处理中，而整个VM的cpu的处理就是在这个循环中不断的进行。

```
int kvm_cpu_exec(CPUArchState *env)
{
    do {
        run_ret = kvm_vcpu_ioctl(env, KVM_RUN, 0);
        switch (run->exit_reason) {
        case KVM_EXIT_IO:
        	kvm_handle_io();
			......
        case KVM_EXIT_MMIO:
        	cpu_physical_memory_rw();
			......
        case KVM_EXIT_IRQ_WINDOW_OPEN:
        	ret = EXCP_INTERRUPT;
            ......
        case KVM_EXIT_SHUTDOWN:
        	ret = EXCP_INTERRUPT;
            ......
        case KVM_EXIT_UNKNOWN:
        	ret = -1
            ......
        case KVM_EXIT_INTERNAL_ERROR:
        	ret = kvm_handle_internal_error(env, run);
            ......
        default:
        	ret = kvm_arch_handle_exit(env, run);
            ......
        }
    } while (ret == 0);
    env->exit_request = 0;
    return ret;
}
```

## Conclusion {#Conclusion}

总结下kvm run在Qemu中的核心流程：

1. 解析参数;
2. 创建三大描述符：kvmfd/vmfd/vcpufd，及相关的初始化，为VM的运行创造必要的条件；
3. 根据cpu的配置数目，启动n个POSIX线程运行VM实体，所以vm的执行环境追根溯源是在Qemu创建的线程环境中开始的。
4. 通过
   _KVM\_RUN_
   调用KVM提供的API发起KVM的启动，从这里进入到了内核空间运行，等待运行返回；
5. 重复循环进入run阶段。

附图：  
![](http://7ktq2s.com1.z0.glb.clouddn.com/2.png)

