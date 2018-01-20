> KVM在I/O虚拟化方面，传统的方式是使用Qemu纯软件的方式来模拟I/O设备，其中包括经常使用的网卡设备。这次我们重点分析Qemu为实现网络设备虚拟化的全虚拟化方案。本主题从三个组成方面来完整描述，包括：1. 前端网络流的建立； 2. 虚拟网卡的创建； 3. 网络I/O虚拟化 in Guest OS。  
> 本篇主要讲述“前端网络流的建立”。

## VM网络配置方式 {#VM网络配置方式}

根据KVM的网络配置方案，大概分为如下几种：

1. 默认用户模式；
2. 基于网桥\(Bridge\)的模式；
3. 基于NAT\(Network Address Translation\)的模式；
4. 网络设备的直接分配\(基于Intel VT-d技术\);

在“kvm安装与启动过程说明”一文中，Guest OS启动命令中没有传入的网络配置,此时QEMU默认分配rtl8139类型的虚拟网卡类型，使用的是默认用户配置模式，这时候由于没有具体的网络模式的配置，Guest的网络功能是有限的。

网桥模式是目前比较简单，也是用的比较多的模式，所以我们这里主要分析基于网桥模式下的VM的收发包流程。

## 网桥的原理与创建 {#网桥的原理与创建}

### 网桥的原理 {#网桥的原理}

网桥\(Bridge\)也称桥接器，是连接两个局域网的存储转发设备，用它可以完成具有相同或相似体系结构网络系统的连接。一般情况下，被连接的网络系统都具有相同的逻辑链路控制规程\(LLC\)，但媒体访问控制协议\(MAC\)可以不同。网桥工作在数据链路层，将两个LAN连起来，根据MAC地址来转发帧，其实就是一个简单的二层交换机。

Linux网络协议栈已经支持了网桥的功能，但需要进行相关的配置才可以进行正常的转发。而要配置Linux网桥功能，需要配置工具bridge-utils，大家可以从网上下载源码编译、安装，生成网桥配置的工具名称为brctl。

关于Linux网桥，这里不再过多的叙述，有兴趣的同学可以自行研究下，其实就是一些MAC地址学习、转发表的维护、及生成树的协议管理等功能，大家就认为它是个实现在内核中的二层交换机就行。

Linux网桥的具体实现可以参看我以前总结的关于网桥的流程分析：[http://blog.csdn.net/hsly\_support/article/details/8762896](http://blog.csdn.net/hsly_support/article/details/8762896)

### 网桥的创建 {#网桥的创建}

```
$brctl addbr br0         #添加br0这个bridge
$brctl addif br0 eth0    #将br0与eth0绑定起来
```

通过上述两条命令，bridge br0成为连接本机与外部网络的接口。

![](http://7ktrxu.com1.z0.glb.clouddn.com/1.png)

## Tap的原理与创建 {#Tap的原理与创建}

qemu-system-x86\_64命令关于bridge模式的网络参数如下：

```
-net tap[,vlan=n][,name=str][,fd=h][,ifname=name][,script=file][,downscript=dfile][,helper=helper][,sndbuf=nbytes][,vnet_hdr=on|off][,vhost=on|off][,vhostfd=h][,vhostforce=on|off]
```

主要参数说明：

```
tap：表示创建一个tap设备；
ifname：表示tap设备接口名字；
script：表示host在启动guest时自动配置的脚本，默认为/etc/qemu-ifup；
downscript：表示host在关闭guest时自动执行的脚本；
fd=h: 连接到现在已经打开着的TAP接口的文件描述符，一般让QEMU会自动创建一个TAP接口;
helper=helper:   设置启动客户机时在宿主机中运行的辅助程序，包括去建立一个TAP虚拟设备，它的默认值为/usr/local/libexec/qemu-bridge-helper，一般不用自定义，采用默认值即可;
sndbuf=nbytes: 限制TAP设备的发送缓冲区大小为n字节，当需要流量进行流量控制时可以设置该选项。其默认值为“sndbuf=0”，即不限制发送缓冲区的大小。
```

### 什么是Tap设备 {#什么是Tap设备}

qemu在这里使用了Tap设备，那Tap设备是什么呢？

TUN/TAP虚拟网络设备的原理比较简单，在Linux内核中添加了一个TUN/TAP虚拟网络设备的驱动程序和一个与之相关连的字符设备/dev/net/tun，字符设备tun作为用户空间和内核空间交换数据的接口。当内核将数据包发送到虚拟网络设备时，数据包被保存在设备相关的一个队列中，直到用户空间程序通过打开的字符设备tun的描述符读取时，它才会被拷贝到用户空间的缓冲区中，其效果就相当于，数据包直接发送到了用户空间。通过系统调用write发送数据包时其原理与此类似。

TUN/TAP驱动程序中包含两个部分，一部分是字符设备驱动，还有一部分是网卡驱动部分。利用网卡驱动部分接收来自TCP/IP协议栈的网络分包并发送或者反过来将接收到的网络分包传给协议栈处理，而字符驱动部分则将网络分包在内核与用户态之间传送，模拟物理链路的数据接收和发送。Tun/tap驱动很好的实现了两种驱动的结合。

总而言之，Tap设备实现了这么一种能力，对于用户空间来讲实现了网络数据包在内核态与用户态之间的传输，对于内核管理来说，它是一个网络设备或者直接呈现的是一个网络接口，可以像普通网络接口一样进行收发包。

### Tap设备的创建 {#Tap设备的创建}

当命令行中通过-net tap指明创建Tap设备后，在Qemu的main函数中首先解析到了tap的参数选项，然后进入了设备创建流程：

```
main()  file: vl.c, line: 2345
    net_init_clients()  file: net.c, line: 991
        net_init_client()  file: net.c, line: 962
            net_client_init()  file: net.c, line: 701
                net_client_init1()  file: net.c, line: 628
                    net_client_init_fun[opts->kind](opts, name, peer)
```

在Qemu中，所有的-net类型都由net client这个概念来表示：net\_client\_init\_fun由各类net client对应的初始化函数组成：

```
static int (* const net_client_init_fun[NET_CLIENT_OPTIONS_KIND_MAX])(
    const NetClientOptions *opts,
    const char *name,
    NetClientState *peer) = {
    	[NET_CLIENT_OPTIONS_KIND_NIC]       = net_init_nic,     <------网卡设备类型
    	[NET_CLIENT_OPTIONS_KIND_TAP]       = net_init_tap,     <------Tap设备类型
    	[NET_CLIENT_OPTIONS_KIND_SOCKET]    = net_init_socket,
    	[NET_CLIENT_OPTIONS_KIND_HUBPORT]   = net_init_hubport,
    };
```

net\_tap\_init\(\)主要做了两件事：

1. 通过tap\_open\(\){open\(“/dev/net/tun”\)}返回了Tap设备的文件描述符fd;
2. 将Tap设备的文件描述符加入Qemu事件监听列表;
3. 通过Qemu-ifup脚本将创建的Tap设备接口加入网桥中,这里假设Tap设备名为tap1；

主要的调用流程如下：

```
net_init_tap()  file: tap.c, line: 589
    net_tap_init()   file: tap.c, line: 552
        tap_open()   file: tap-linux.c, line: 38
            fd = open(PATH_NET_TUN, O_RDWR)   file: tap-linux.c, line: 43
    net_tap_fd_init()   file: tap.c, line: 325
        tap_read_poll()  file: tap.c, line: 81
            tap_update_fd_handler()  file: tap.c, line: 72
                qemu_set_fd_handler2()  file: iohandler.c, line: 51
                    QLIST_INSERT_HEAD(&io_handlers, ioh, next);   file: iohandler.c, line: 72
```

可以看到最后Tap设备的事件通知加入了io\_handlers的事件监听列表中，fd\_read事件对应的动作为tap\_send\(\),fd\_write事件对应的动作为tap\_writable\(\)。

Tap设备接口加入网桥命令：

```
$brctl addif br0 tap1
```

![](http://7ktrxu.com1.z0.glb.clouddn.com/2.png)

### Qemu主线程中的事件监听 {#Qemu主线程中的事件监听}

io\_handlers的事件监听在哪里发生呢？

Qemu的Main函数通过一系列的初始化，并创建线程进行VM的启动，最后来到了main\_loop\(\)\(file: vl.c, line: 3790\)

```
main_loop()  file: vl.c, line: 1631
    main_loop_wait()  file: main-loop.c, line: 473
        qemu_iohandler_fill()  file: io_handler.c, line: 93
        os_host_main_loop_wait()  file: main-loop.c, line: 291
            select(nfds + 1, &rfds, &wfds, &xfds, tvarg)   <----此处对注册的源进行监听，包括Tap fd;
        qemu_iohandler_poll()  file: io_handler.c, line: 115 <---调用事件对应动作进行处理；
```

## 前端网络数据流程 {#前端网络数据流程}

![](http://7ktrxu.com1.z0.glb.clouddn.com/3.png)

如图中所示，红色箭头表示数据报文的入方向，步骤：

1. 网络数据从Host上的物理网卡接收，到达网桥；
2. 由于eth0与tap1均加入网桥中，根据二层转发原则，br0将数据从tap1口转发出去，即数据由Tap设备接收；
3. Tap设备通知对应的fd数据可读；
4. fd的读动作通过tap设备的字符设备驱动将数据拷贝到用户空间，完成数据报文的前端接收。

绿色箭头表示数据报文的出方向，步骤与入方向相反,这里不再详细叙述。

