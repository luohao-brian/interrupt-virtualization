> KVM上设备I/O虚拟化的性能问题长期存在，此时由Rusty Russell开发的virtio引起了开发者们的注意并逐渐被KVM等虚拟化平台接纳并作为了其I/O虚拟化最主要的一个通用框架。
>
> Virtio使用virtqueue来实现其I/O机制，每个virtqueue就是一个承载大量数据的queue。vring是virtqueue的具体实现方式。



## Virtqueue：传输层的抽象 {#Virtqueue：传输层的抽象}

每个设备拥有多个virtqueue用于大块数据的传输。virtqueue是一个简单的队列，Guest把buffers插入其中，每个buffer 都是一个分散-聚集数组。驱动调用find\_vqs\(\)来创建一个与queue关联的结构体。virtqueue的数目根据设备的不同而不同，比如block设备有一个virtqueue,network设备有2个virtqueue,一个用于发送数据包，一个用于接收数据包。

在virtio设备创建过程中，形成的数据结构如图所示：

![](http://7lrywd.com1.z0.glb.clouddn.com/2.PNG)

从图中可以看出，virtio-netdev关联了两个virtqueue，包括一个send queue和一个receive queue，而具体的queue的实现由vring来承载。

针对 virtqueue 的操作包括：

```
int virtqueue_add_buf( struct virtqueue *_vq, struct scatterlist sg[], unsigned int out, unsigned int in, void *data, gfp_t gfp)
```

add\_buf\(\)用于向queue中添加一个新的buffer，参数data是一个非空的令牌，用于识别 buffer，当buffer内容被消耗后，data会返回。

```
virtqueue_kick()
```

Guest 通知 host 单个或者多个 buffer 已经添加到 queue 中,调用 virtqueue\_notify\(\)，notify 函数会向 queue notify\(VIRTIO\_PCI\_QUEUE\_NOTIFY\)寄存器写入 queue index 来通知 host。  


```
void *virtqueue_get_buf(struct virtqueue *_vq, unsigned int *len)
```

返回使用过的buffer，len为写入到buffer中数据的长度。获取数据，释放buffer,更新vring描述符表格中的index。  


```
virtqueue_disable_cb()
```

Guest不再需要知道一个buffer已经使用了，也就是关闭device的中断。驱动会在初始化时注册一个回调函数，disable\_cb\(\)通常在这个virtqueue回调函数中使用，用于关闭再次的回调函数调用。  


```
virtqueue_enable_cb()
```

与 disable\_cb\(\)刚好相反，用于重新开启设备中断的上报。

## vring的创建流程 {#vring的创建流程}

在virtio netdev driver被加载后，会调用virtnet\_probe进行设备的识别，创建，初始化。

```
static struct virtio_driver virtio_net_driver = {
    .feature_table = features,
    .feature_table_size = ARRAY_SIZE(features),
    .driver.name =    KBUILD_MODNAME,
    .driver.owner =    THIS_MODULE,
    .id_table =    id_table,
    .probe =    virtnet_probe,  <------------识别，初始化入口
    .remove =    virtnet_remove,
    .config_changed = virtnet_config_changed,
    #ifdef CONFIG_PM_SLEEP
    .freeze =    virtnet_freeze,
    .restore =    virtnet_restore,
    #endif
};
```

其中进行了virtio net device的属性功能的配置，网络设备的初始化和注册，而vring的创建也在其中：

```
/* Allocate/initialize the rx/tx queues, and invoke find_vqs */
init_vqs(vi);          //创建和初始化发送/接收队列
    --->virtnet_alloc_queues()
    --->virtnet_find_vqs()
```

virtnet\_alloc\_queues创建了图中virtnet\_info中的send\_queue 和 receive\_queue结构，seng和receive queue是成对出现的。

```
static int virtnet_alloc_queues(struct virtnet_info *vi) 
{
    int i;
     vi->sq = kzalloc(sizeof(*vi->sq) * vi->max_queue_pairs, GFP_KERNEL);
     if (!vi->sq)
          goto err_sq;
    vi->rq = kzalloc(sizeof(*vi->rq) * vi->max_queue_pairs, GFP_KERNEL);
     if (!vi->rq)
          goto err_rq;
     INIT_DELAYED_WORK(&vi->refill, refill_work);
     for (i = 0; i < vi->max_queue_pairs; i++) {
          vi->rq[i].pages = NULL;
          netif_napi_add(vi->dev, &vi->rq[i].napi, virtnet_poll, napi_weight);
          sg_init_table(vi->rq[i].sg, ARRAY_SIZE(vi->rq[i].sg)); //初始化收端的scatterlist
          ewma_init(&vi->rq[i].mrg_avg_pkt_len, 1, RECEIVE_AVG_WEIGHT);
          sg_init_table(vi->sq[i].sg, ARRAY_SIZE(vi->sq[i].sg)); //初始化发端的scatterlist 
     }
     return 0;
err_rq:
    kfree(vi->sq);
    err_sq:
     return -ENOMEM;
}
```

scatterlist即是一种数组形式的数据结构，每一项成员指向一个page的地址，偏移量，长度等。  
![](http://7lrywd.com1.z0.glb.clouddn.com/3.png)

通过find vqs来创建vring:

```
static int virtnet_find_vqs(struct virtnet_info *vi)  
{
    ......
    vi->vdev->config->find_vqs(vi->vdev, total_vqs, vqs, callbacks, names);
    ......
}
```

vdev所对应的config的初始化在pci bus probe阶段:

```
static int virtio_pci_probe(struct pci_dev *pci_dev, const struct pci_device_id *id)
{
    ......
    vp_dev->vdev.config = &virtio_pci_config_ops;
    ......
}
```

virtio\_pci\_config\_ops是针对virtio设备配置的操作方法，主要包括四个部分1. 读写特征位;2. 读写配置空间;3. 读写状态位;4. 重启设备.

```
static const struct virtio_config_ops virtio_pci_config_ops = {
    .get     = vp_get,                    //读取virtio配置空间的域
    .set     = vp_set,                    //设置virtio配置空间的域
    .get_status    = vp_get_status,          //读取状态位
    .set_status    = vp_set_status,          //设置状态位
    .reset     = vp_reset,                  //设备的复位
    .find_vqs    = vp_find_vqs,            //virtqueue的创建
    .del_vqs    = vp_del_vqs,             //virtqueue的删除
    .get_features    = vp_get_features,
    .finalize_features = vp_finalize_features,
    .bus_name    = vp_bus_name,
    .set_vq_affinity = vp_set_vq_affinity,
};
```

其中最核心的是setup\_vq\(\)：

```
/*
    函数功能：为目标设备获取一个queue
    vdev：目标设备
    index：给目标设备使用的queue的编号
    callback：queue的回调函数
    name：queue的名字
    msix_vec：给queue使用的msix vector的编号
*/
static struct virtqueue *setup_vq(struct virtio_device *vdev, unsigned index,
   void (*callback)(struct virtqueue *vq),
   const char *name,
   u16 msix_vec)
{
    //通过VIRTIO_PCI_QUEUE_SEL配置域，选择我们需要的queue的编号index
    iowrite16(index, vp_dev->ioaddr + VIRTIO_PCI_QUEUE_SEL);

    //通过读取VIRTIO_PCI_QUEUE_NUM配置域，获取index编号的queue的
    num = ioread16(vp_dev->ioaddr + VIRTIO_PCI_QUEUE_NUM); 

     /* 如果num为0，则该queue不可用
       通过读取VIRTIO_PCI_QUEUE_PFN配置域返回queue的地址，
       如果该queue的地址非空，则说明已经在被使用了，该queue不可用。
      */
     if (!num || ioread32(vp_dev->ioaddr + VIRTIO_PCI_QUEUE_PFN))
          return ERR_PTR(-ENOENT);

    info = kmalloc(sizeof(struct virtio_pci_vq_info), GFP_KERNEL);

    //计算vring所需要的空间大小
    size = PAGE_ALIGN(vring_size(num, VIRTIO_PCI_VRING_ALIGN));  

     //分配size大小的若干个page空间为vring所用
     info->queue = alloc_pages_exact(size, GFP_KERNEL|__GFP_ZERO);    

    /* activate the queue
       通过VIRTIO_PCI_QUEUE_PFN配置域将vring的地址通告给qemu，
       这样index编号的queue有大小，有空间qemu通过该块vring的
       共享空间与guest进行数据的交互
    */
     iowrite32(virt_to_phys(info->queue) >> VIRTIO_PCI_QUEUE_ADDR_SHIFT,
        vp_dev->ioaddr + VIRTIO_PCI_QUEUE_PFN);  

    /* create the vring */
    //对vring内部结构进行具体的初始化
     vq = vring_new_virtqueue(index, info->num, VIRTIO_PCI_VRING_ALIGN, vdev,
         true, info->queue, vp_notify, callback, name); 
}
```

关于num的初始值，在qemu初始化virtionet设备时进行了初始化，queue size 即kernel中读取的num初始化为256个\(file:hw/virtio-net.c, line:999\):

```
VirtIODevice *virtio_net_init(DeviceState *dev, NICConf *conf, virtio_net_conf *net)
{
    ...
    n->rx_vq = virtio_add_queue(&n->vdev, 256, virtio_net_handle_rx);
    ...
    n->tx_vq = virtio_add_queue(&n->vdev, 256, virtio_net_handle_tx_bh);
    ...
}
```

virtio\_ring的具体结构包含3部分：

1. 描述符数组\(descriptor table\)用于存储一些关联的描述符，每个描述符都是一个对buffer的描述，包含一个 address/length的配对。
2. 可用的ring\(available ring\)用于Guest端表示那些描述符链当前是可用的。
3. 使用过的ring\(used ring\)用于表示Host端表示那些描述符已经使用。

Ring的数目必须是2的次幂。结构如下图所示：

![](http://7lrywd.com1.z0.glb.clouddn.com/4.PNG)

vring descriptor 用于指向 guest 使用的 buffer。  
addr：guest 物理地址  
len：buffer 的长度  
flags：flags 的值含义包括：

* VRING\_DESC\_F\_NEXT：用于表明当前 buffer 的下一个域是否有效，也间接表明当前 buffer 是否是 buffers list 的最后一个。
* VRING\_DESC\_F\_WRITE：当前 buffer 是 read-only 还是 write-only。
* VRING\_DESC\_F\_INDIRECT：表明这个 buffer 中包含一个 buffer 描述符的 list

next：所有的 buffers 通过next串联起来组成descriptor table  
多个buffer组成一个list由descriptor table指向这些list。

Available ring 指向 guest 提供给设备的描述符,它指向一个 descriptor 链表的头。

Used ring 指向 device\(host\)使用过的 buffers。

