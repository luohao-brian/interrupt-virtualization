> 在前面几文中已经大体介绍了virtio的重要组成，包括virtio net设备的创建，vring的创建，与virtio设备的交互方式，我们就从网络数据包的发送角度来看下virtio的具体使用流程。

## 流程分析 {#流程分析}

当Kernel中的网络数据包从内核协议栈下来后，必然要走到设备注册的发送函数， virtio netdev注册的的发送函数即virtnet\_netdev中的virtnet\_netdev\(\);

```
static const struct net_device_ops virtnet_netdev = {
	.ndo_open = virtnet_open,
	.ndo_stop = virtnet_close,
	.ndo_start_xmit = start_xmit,        <----------------
	.ndo_validate_addr = eth_validate_addr,
	.ndo_set_mac_address = virtnet_set_mac_address,
	.ndo_set_rx_mode = virtnet_set_rx_mode,
	.ndo_change_mtu	 = virtnet_change_mtu,
	.ndo_get_stats64 = virtnet_stats,
	.ndo_vlan_rx_add_vid = virtnet_vlan_rx_add_vid,
	.ndo_vlan_rx_kill_vid = virtnet_vlan_rx_kill_vid,
	#ifdef CONFIG_NET_POLL_CONTROLLER
 	.ndo_poll_controller = virtnet_netpoll,
	#endif
};
static netdev_tx_t start_xmit(struct sk_buff *skb, struct net_device *dev)
{
	......
    err = xmit_skb(sq, skb);  
	......
    virtqueue_kick(sq->vq);
	......
}
```

在start\_xmit\(\)中，主要的操作是使数据包入vring队列：

```
static int xmit_skb(struct send_queue *sq, struct sk_buff *skb)
{
    struct skb_vnet_hdr *hdr;
    hdr_len = sizeof hdr->hdr;
    hdr = skb_vnet_hdr(skb);
    ......
    sg_set_buf(sq->sg, hdr, hdr_len);
  	num_sg = skb_to_sgvec(skb, sq->sg + 1, 0, skb->len) + 1;
    return virtqueue_add_outbuf(sq->vq, sq->sg, num_sg, skb, GFP_ATOMIC);
}
```

在每个进入scatter-gather list的packet之前，需要有一个virtio\_net\_hdr结构的头部信息，用以  
支持checksum offload与TCP/UDP Segmentation offload。所以在上述流程中先使用sg\_set\_buf\(sq-&gt;sg, hdr, hdr\_len\)将virtio-net-hdr的buffer填入了scatter-gather list，如下是virtio\_net\_hdr的结构;

```
struct skb_vnet_hdr {
	union {
		struct virtio_net_hdr hdr;
		struct virtio_net_hdr_mrg_rxbuf mhdr;
	};
};
struct virtio_net_hdr {
	#define VIRTIO_NET_HDR_F_NEEDS_CSUM	1	// Use csum_start, csum_offset
	#define VIRTIO_NET_HDR_F_DATA_VALID	2	// Csum is valid
 	__u8 flags;
	#define VIRTIO_NET_HDR_GSO_NONE	 0	// Not a GSO frame
	#define VIRTIO_NET_HDR_GSO_TCPV4	1	// GSO frame, IPv4 TCP (TSO)
	#define VIRTIO_NET_HDR_GSO_UDP	 3	// GSO frame, IPv4 UDP (UFO)
	#define VIRTIO_NET_HDR_GSO_TCPV6	4	// GSO frame, IPv6 TCP
	#define VIRTIO_NET_HDR_GSO_ECN	 0x80	// TCP has ECN set
 	__u8 gso_type;
 	__u16 hdr_len;	 /* Ethernet + IP + tcp/udp hdrs */
 	__u16 gso_size;	 /* Bytes to append to hdr_len per frame */
 	__u16 csum_start;	/* Position to start checksumming from */
 	__u16 csum_offset;	/* Offset after that to place checksum */
};
```

将virtio\_net\_hdr塞入scatter-gather list，然后再入packet的buffer。其中都会调用到sg\_set\_page\(\)，主要的操作就是计算待发送数据buffer占用的page的基址，相对基址的偏移量及length。

```
static inline void sg_set_buf(struct scatterlist *sg, const void *buf, unsigned int buflen)
{
 	sg_set_page(sg, virt_to_page(buf), buflen, offset_in_page(buf));
}
static inline void sg_set_page(struct scatterlist *sg, struct page *page,
          unsigned int len, unsigned int offset)
{
 	sg_assign_page(sg, page);
 	sg->offset = offset;
 	sg->length = len;
}
static inline void sg_assign_page(struct scatterlist *sg, struct page *page)
{
 	unsigned long page_link = sg->page_link & 0x3;
	/*
	 * In order for the low bit stealing approach to work, pages
	 * must be aligned at a 32-bit boundary as a minimum.
	 */
	BUG_ON((unsigned long) page & 0x03);
	#ifdef CONFIG_DEBUG_SG
	BUG_ON(sg->sg_magic != SG_MAGIC);
	BUG_ON(sg_is_chain(sg));
	#endif
	sg->page_link = page_link | (unsigned long) page;
}
```

![](http://7lrywd.com1.z0.glb.clouddn.com/5.PNG)

skbuffer与sg list的关系如上所示。

最后调用return virtqueue\_add\_outbuf\(sq-&gt;vq, sq-&gt;sg, num\_sg, skb, GFP\_ATOMIC\);进入vring操作阶段。

```
/**
 * virtqueue_add_outbuf - expose output buffers to other end
 * @vq: the struct virtqueue we're talking about.
 * @sgs: array of scatterlists (need not be terminated!)
 * @num: the number of scatterlists readable by other side
 * @data: the token identifying the buffer.
 * @gfp: how to do memory allocations (if necessary).
 *
 * Caller must ensure we don't call this with other virtqueue operations
 * at the same time (except where noted).
 *
 * Returns zero or a negative error (ie. ENOSPC, ENOMEM).
 */
int virtqueue_add_outbuf(struct virtqueue *vq,
    struct scatterlist sg[], unsigned int num,
    void *data,
    gfp_t gfp)
{
 	return virtqueue_add(vq, &sg, sg_next_arr, num, 0, 1, 0, data, gfp);
}
static inline int virtqueue_add(struct virtqueue *_vq,
    struct scatterlist *sgs[],
    struct scatterlist *(*next)
      (struct scatterlist *, unsigned int *),
    unsigned int total_out,
    unsigned int total_in,
    unsigned int out_sgs,
    unsigned int in_sgs,
    void *data,
    gfp_t gfp)
{
   	......
	head = i = vq->free_head;
   	for (n = 0; n < out_sgs; n++) {
 			for (sg = sgs[n]; sg; sg = next(sg, &total_out)) {
  				vq->vring.desc[i].flags = VRING_DESC_F_NEXT;
  				vq->vring.desc[i].addr = sg_phys(sg);
  				vq->vring.desc[i].len = sg->length;
  				prev = i;
  				i = vq->vring.desc[i].next; //通过next字段找到下一个可用的desc
 			}
		}

   	/* Last one doesn't continue. */
		vq->vring.desc[prev].flags &= ~VRING_DESC_F_NEXT; 

	/* Update free pointer */
		vq->free_head = i;

   	/* Set token. */
		vq->data[head] = data;

   	/* Put entry in available array (but don't update avail->idx until they do sync). */
		avail = (vq->vring.avail->idx & (vq->vring.num-1));
		vq->vring.avail->ring[avail] = head;
		/* Descriptors and available array need to be set before we expose the new available array entries. */
		virtio_wmb(vq->weak_barriers);
		vq->vring.avail->idx++;
		vq->num_added++;
}
```

1. 从head = i = vq-
   &gt;
   free\_head;找到第一片可用的desc;
2. 从sg list将需要发送buffer信息读取并填充vring的desc描述符;  
   addr: guest的物理地址  
   len： buffer的长度  
   flags： VRING\_DESC\_F\_NEXT表示该片buffer还有后续片

3. 将最后一片占用的desc的flag作下标记，表示buffer片的终结;

4. 更新空闲desc的指针;

5. 将skb保存在data\[\]中作为token，用完后再释放；

6. 更新avail描述符，将待发送的第一片buffer在desc中的序号写入空闲的avail ring中，并更新avail描述队列的序号等。

网上一幅图可以看到这些操作的关系：

![](http://7lrywd.com1.z0.glb.clouddn.com/6.PNG)

在start\_xmit中，待发送的信息入队列后，使用virtqueue\_kick\(sq-&gt;vq\)通告Host端；

```
bool virtqueue_kick(struct virtqueue *vq)
{
 	if (virtqueue_kick_prepare(vq))
  		return virtqueue_notify(vq);
 	return true;
}
bool virtqueue_notify(struct virtqueue *_vq)
{
 	struct vring_virtqueue *vq = to_vvq(_vq);
 	if (unlikely(vq->broken))
  		return false;
 	/* Prod other side to tell it about changes. */
 	if (!vq->notify(_vq)) {
  		vq->broken = true;
  		return false;
 	}
 	return true;
}
```

其中vq-&gt;notifynotify即是vring创建时注册的vp\_notify。

```
static bool vp_notify(struct virtqueue *vq)
{
 	struct virtio_pci_device *vp_dev = to_vp_device(vq->vdev);
 	/* we write the queue's selector into the notification register to
  	* signal the other end */
 	iowrite16(vq->index, vp_dev->ioaddr + VIRTIO_PCI_QUEUE_NOTIFY);
 	return true;
}
```

通过配置VIRTIO\_PCI\_QUEUE\_NOTIFY域来进行通告。接下来就是HOST一端的处理了。从整个前端发送流程可以看出，一个数据包发送时只是将skb的地址及长度等信息通告了virtio driver，而vring的空间是和后端共享的，所以该传输过程为零拷贝，这也是virtio高性能的一个原因。

