chttps://blog.csdn.net/wangquan1992/article/details/120649182



# 一、virtio架构层次

![0](https://img-blog.csdnimg.cn/img_convert/da4084dea640d35d8e3386d95a301db7.png)

### 1.virtio前端驱动

① 运行在虚拟机中

② 针对不同类型的设备有不同的驱动程序，但是与后端驱动交互的接口都是统一的

③ virtio-net模块，源码位于drivers/net/virtio_net.c

### 2.virtio层

① **virtio层实现虚拟队列接口**，作为前后端通信的桥梁

② 不同类型的设备使用的虚拟队列数量不同，比如virtio-net使用两个队列，一个用于接收，另一个用于发送

③ 源码位于drivers/virtio/virtio.c

### 3.virtio-ring层

① **virtio-ring层是虚拟队列的具体实现**

② 源码位于driver/virtio/virtio_ring.c

### 4.virtio后端驱动

① 运行在宿主机中

② 实现virtio后端的逻辑，主要是操作硬件设备，比如向内核协议栈发送一个网络包完成虚拟机对于网络的操作

③ 在Qemu + KVM虚拟化环境中，源码位于Qemu源码中。



# 二、virtio核心数据结构

### 1、virtio_bus结构

bus_type是基于总线驱动模型的公共数据结构，定义新的bus，就是填充该结构

定义在drivers/virtio/virtio.c中

```c
static struct bus_type virtio_bus = {
	.name  = "virtio",
	.match = virtio_dev_match, //匹配函数
	.dev_groups = virtio_dev_groups,
	.uevent = virtio_uevent,
	.probe = virtio_dev_probe,
	.remove = virtio_dev_remove,
};
```

#### （1）注册virtio_bus

```c
static int virtio_init(void)
{
	if (bus_register(&virtio_bus) != 0) //bus注册
		panic("virtio bus registration failed");
	return 0;
}

static void __exit virtio_exit(void)
{
	bus_unregister(&virtio_bus);
	ida_destroy(&virtio_index_ida);
}
core_initcall(virtio_init); //#define core_initcall(fn)		__define_initcall(fn, 1) //include/linux/init.h
module_exit(virtio_exit);
```

virtio_bus以core_initcall方式注册，启动顺序优先级高

### （2）virtio_dev_match

```c
/* This looks through all the IDs a driver claims to support.  If any of them
 * match, we return 1 and the kernel will call virtio_dev_probe(). */
static int virtio_dev_match(struct device *_dv, struct device_driver *_dr)
{
	unsigned int i;
	struct virtio_device *dev = dev_to_virtio(_dv); //---1
	const struct virtio_device_id *ids;

	ids = drv_to_virtio(_dr)->id_table;//---2
	for (i = 0; ids[i].device; i++)
		if (virtio_id_match(dev, &ids[i]))//---3
			return 1;
	return 0;
}
```

virtio_device_id结构体相关：

virtio_device结构体中包含virtio_device_id；

virtio_driver结构体中包含该驱动支持的virtio_device_id列表

#### ---1

根据device结构索引virtio_device结构；

==》device结构转换为virtio_device结构

```c
static inline struct virtio_device *dev_to_virtio(struct device *_dev)
{
	return container_of(_dev, struct virtio_device, dev);
}
/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:	the pointer to the member.
 * @type:	the type of the container struct this is embedded in.
 * @member:	the name of the member within the struct.
 *
 */
//结构体type的成员member的地址ptr，求解结构体type的起始地址
#define container_of(ptr, type, member) ({			\
	const typeof( ((type *)0)->member ) *__mptr = (ptr);	\ 	
	(type *)( (char *)__mptr - offsetof(type,member) );}) 		
```

#### --2

根据device_driver结构索引virtio_driver结构，并取出其中的id_table；

```c
static inline struct virtio_driver *drv_to_virtio(struct device_driver *drv)
{
	return container_of(drv, struct virtio_driver, driver);
}
```

#### ---3

根据id_table（struct virtio_device_id类型）进行匹配；

先匹配device，后匹配vendor，两者都满足时match成功；

```c
static inline int virtio_id_match(const struct virtio_device *dev,
				  const struct virtio_device_id *id)
{
	if (id->device != dev->id.device && id->device != VIRTIO_DEV_ANY_ID)
		return 0;

	return id->vendor == VIRTIO_DEV_ANY_ID || id->vendor == dev->id.vendor;
}
```

### 2、virtio_device结构

```c
struct virtio_device {
	int index;
	bool failed;
	bool config_enabled;
	bool config_change_pending;
	spinlock_t config_lock;
	struct device dev;
	struct virtio_device_id id;//(1)
	const struct virtio_config_ops *config;//(2)
	const struct vringh_config_ops *vringh_config;
	struct list_head vqs;//(3)
	u64 features;//(4)
	void *priv;
};
```

#### （1）struct virtio_device_id

device标记了virtio_device的类型，类似的有virtio-net，virtio-blk等

```c
//include/linux/mod_devicetable.h
struct virtio_device_id {
	__u32 device;
	__u32 vendor;
};
#define VIRTIO_DEV_ANY_ID	0xffffffff
```

```c
//include/linux/virtio_ids.h
#define VIRTIO_ID_NET		1 /* virtio net */
#define VIRTIO_ID_BLOCK		2 /* virtio block */
#define VIRTIO_ID_CONSOLE	3 /* virtio console */
#define VIRTIO_ID_RNG		4 /* virtio rng */
#define VIRTIO_ID_BALLOON	5 /* virtio balloon */
#define VIRTIO_ID_RPMSG		7 /* virtio remote processor messaging */
#define VIRTIO_ID_SCSI		8 /* virtio scsi */
#define VIRTIO_ID_9P		9 /* 9p virtio console */
#define VIRTIO_ID_RPROC_SERIAL 11 /* virtio remoteproc serial link */
#define VIRTIO_ID_CAIF	       12 /* Virtio caif */
#define VIRTIO_ID_GPU          16 /* virtio GPU */
#define VIRTIO_ID_INPUT        18 /* virtio input */
#define VIRTIO_ID_VSOCK        19 /* virtio vsock transport */
#define VIRTIO_ID_CRYPTO       20 /* virtio crypto */

static struct virtio_device_id id_table[] = {
	{ VIRTIO_ID_NET, VIRTIO_DEV_ANY_ID }, //device vendor
	{ 0 },
};
```

#### （2）struct virtio_config_ops

```c
typedef void vq_callback_t(struct virtqueue *);
struct virtio_config_ops {
	void (*get)(struct virtio_device *vdev, unsigned offset, //获取virtio_device的属性和状态
		    void *buf, unsigned len);
	void (*set)(struct virtio_device *vdev, unsigned offset, //设置virtio_device的属性和状态
		    const void *buf, unsigned len);
	u32 (*generation)(struct virtio_device *vdev);
	u8 (*get_status)(struct virtio_device *vdev);
	void (*set_status)(struct virtio_device *vdev, u8 status);
	void (*reset)(struct virtio_device *vdev);
	int (*find_vqs)(struct virtio_device *, unsigned nvqs,
			struct virtqueue *vqs[], vq_callback_t *callbacks[],
			const char * const names[], const bool *ctx,
			struct irq_affinity *desc); //用于实例化virtio_device所持有的virtqueue
	void (*del_vqs)(struct virtio_device *);
	u64 (*get_features)(struct virtio_device *vdev);
	int (*finalize_features)(struct virtio_device *vdev);
	const char *(*bus_name)(struct virtio_device *vdev);
	int (*set_vq_affinity)(struct virtqueue *vq, int cpu);
	const struct cpumask *(*get_vq_affinity)(struct virtio_device *vdev,
			int index);
};
```

以下函数注册钩子：

#### //doing

```c
virtio_pci_config_ops in virtio_pci_legacy.c (linux-4.18\drivers\virtio) : 	.find_vqs	= vp_find_vqs,
virtio_pci_config_nodev_ops in virtio_pci_modern.c (linux-4.18\drivers\virtio) : 	.find_vqs	= vp_modern_find_vqs,
virtio_pci_config_ops in virtio_pci_modern.c (linux-4.18\drivers\virtio) : 	.find_vqs	= vp_modern_find_vqs,
```

#### （3）struct list_head vqs

virtio_device持有的virtqueue链表，virtio-net中建立了2条virtqueue，见以下流程：

```c
//先加载virtio-pci，后加载virtio-net
#obj-$(CONFIG_VIRTIO)        += virtio/
#obj-y               += net/

virtnet_probe
	err = init_vqs(vi);
		ret = virtnet_find_vqs(vi);
			callbacks[rxq2vq(i)] = skb_recv_done;
			callbacks[txq2vq(i)] = skb_xmit_done;
			vi->vdev->config->find_vqs(vi->vdev, total_vqs, vqs, callbacks, names, ctx, NULL)
				vp_modern_find_vqs
					vp_find_vqs
						vp_find_vqs_msix
                			//为rx,tx各创建virtqueue
							vp_setup_vq
								vq = vp_dev->setup_vq(vp_dev, info, index, callback, name, ctx, msix_vec);
									vring_create_virtqueue
```

#### （4）u64 features

virtio_driver（feature_table） & virtio_device同时支持的通信特性，也就是前后端最终协商的通信特性



### 3、virtio_driver结构

```c
struct virtio_driver {
	struct device_driver driver;
	const struct virtio_device_id *id_table;//(1)
	const unsigned int *feature_table;//(2)
	unsigned int feature_table_size;//(2)
	const unsigned int *feature_table_legacy;
	unsigned int feature_table_size_legacy;
	int (*validate)(struct virtio_device *dev);
	int (*probe)(struct virtio_device *dev);//(3)
	void (*scan)(struct virtio_device *dev);
	void (*remove)(struct virtio_device *dev);
	void (*config_changed)(struct virtio_device *dev);
#ifdef CONFIG_PM
	int (*freeze)(struct virtio_device *dev);
	int (*restore)(struct virtio_device *dev);
#endif
};
```

#### （1）const struct virtio_device_id

对应virtio_device结构中的id成员，virtio_device中标识的是当前device的id属性；

而virtio_driver中的id_table则是当前driver支持的所有id列表

#### （2）feature_table & feature_table_size

feature列表包含了当前driver支持的所有virtio传输属性，feature_table_size则说明属性数组的元素个数

#### （3）probe

在Linux总线驱动框架 & virtio核心层中，当virtio_device & virtio_driver匹配成功后，先调用bus层面的probe函数，在virtio_bus层面的probe函数中，又调用了virtio_driver层面的probe函数

```c
//pci设备match
//PCI设备注册后，驱动核心遍历总线上的每个驱动，依次执行match函数，查看是否能匹配
__pci_device_probe
    id = pci_match_device(drv, pci_dev);
	if (id) 
		error = pci_call_probe(drv, pci_dev, id); //匹配成功调用probe函数
    
virtio_dev_probe //virtio_bus
    drv->probe(dev);
		virtnet_probe //virtio_driver
```

### 4、virtqueue数据结构

```c
struct virtqueue {
	struct list_head list;
	void (*callback)(struct virtqueue *vq); //virtqueue被触发中断时执行的回调
	const char *name;
	struct virtio_device *vdev; //virtqueue所属virtio_device
	unsigned int index;
	unsigned int num_free; //virtqueue中空闲的描述符个数
	void *priv;
};
```

### 5、vring结构

```c
struct vring {
	unsigned int num;
	struct vring_desc *desc; //描述符table；描述内存buffer，主要包括addr,len等
	struct vring_avail *avail; //前端通知后端有可用描述符；前端有数据发送，就放到avail，后端去取
	struct vring_used *used; //后端通知前端有可用描述符；后端有数据发送，就放到used，前端去取
};

/* Virtio ring descriptors: 16 bytes.  These can chain together via "next". */
struct vring_desc {
	/* Address (guest-physical). */
	__virtio64 addr;
	/* Length. */
	__virtio32 len;
	/* The flags as indicated above. */
	__virtio16 flags;
	/* We chain unused descriptors via this, too */
	__virtio16 next;
};

struct vring_avail {
	__virtio16 flags;
	__virtio16 idx;
	__virtio16 ring[];
};

struct vring_used {
	__virtio16 flags;
	__virtio16 idx;
	struct vring_used_elem ring[];
};

/* u32 is used here for ids for padding reasons. */
struct vring_used_elem {
	/* Index of start of used descriptor chain. */
	__virtio32 id;
	/* Total length of the descriptor chain which was used (written to) */
	__virtio32 len;
};
```

#### （1）vring内存布局

vring的三个区域是在内存中连续存储的，而且是存储在Guest & Host共享的一片连续内存中

![0](https://img-blog.csdnimg.cn/img_convert/ce6065a51d00feb909417b3d66c71397.png)

计算used ring的起始地址时，在avail->ring[num]的地址之后又加了sizeof(__virtio16)，也就是增加了2B，是为了容纳avail ring末尾的used_event（是一种控制中断触发频率的机制）

```c
static inline void vring_init(struct vring *vr, unsigned int num, void *p,
			      unsigned long align)
{
	vr->num = num;
	vr->desc = p;
	vr->avail = p + num*sizeof(struct vring_desc);
	vr->used = (void *)(((uintptr_t)&vr->avail->ring[num] + sizeof(__virtio16) + align-1) & ~(align - 1));
}
```

#### （2）vring实际大小

```c
static inline unsigned vring_size(unsigned int num, unsigned long align)
{
	return ((sizeof(struct vring_desc) * num + sizeof(__virtio16) * (3 + num) + align - 1) & ~(align - 1))
		+ sizeof(__virtio16) * 3 + sizeof(struct vring_used_elem) * num;
}
```

① 计算avail ring时加3，分别为flags、idx和used_event

② 计算used ring时加3，分别为flags、idx和avail_event

③ 计算过程中，包含了为满足对齐要求padding的空间

#### （3）used_event与avail_event机制概述

这2个字段均与virtio设备的VIRTIO_RING_F_EVENT_IDX特性有关，由于virtio驱动触发对方中断将导致CPU反复进出虚拟机 & 宿主机模式，从而降低性能，因此需要控制触发中断频率的机制

```c
//用来对前后端速率进行匹配限速
__vring_new_virtqueue
    //vq->event初始化
    vq->event = virtio_has_feature(vdev, VIRTIO_RING_F_EVENT_IDX);

start_xmit
    virtqueue_kick
        virtqueue_kick_prepare
            if (vq->event) {
                needs_kick = vring_need_event(virtio16_to_cpu(_vq->vdev, vring_avail_event(&vq->vring)), new, old);
            } else {
                needs_kick = !(vq->vring.used->flags & cpu_to_virtio16(_vq->vdev, VRING_USED_F_NO_NOTIFY));
            }	    
```

#### （4）avail ring作用

一是发送侧(send queue)前端驱动发送报文的时，将待发送报文加入avail ring等待后端的处理，后端处理完后，会将其放入used ring，并由前端将其释放desc中(free_old_xmit_skbs, detach_buf)，最后通过try_fill_recv重新装入avail ring中; 

二是接收侧(receive queue)，前端将<b style="color:red;">空白物理块</b>加入avail ring中，提供给后端用来接收报文，后端接收完报文会放入used ring

可以看出：都是后端用完前端的avail ring的东西放入used ring，也就是前端消耗used，后端消耗avail。所以本特性中后端用了used ring的最后一个元素，告诉前端驱动后端处理到哪个avail ring上的元素了，同时前端使用avail ring的最后一个元素告诉后端，处理到那个used ring了。



① avail ring中的used_event

a. 由前端驱动（Geust）设置，标识希望后端驱动（Host）触发中断的阈值

b. 后端驱动（Host）在向Used Ring加入buffer后，检查Used Ring中的idx字段，只有达到阈值才触发中断

```c
//前端通过NAPI接收数据时，会在可用buffer不足的时候调用函数添加
virtnet_poll
    virtnet_receive
    	//取出skb，释放对应desc
    	virtqueue_get_buf_ctx
    	//根据支持的特性调用不同的函数构建skb
    	receive_buf
    		//发往协议栈
    		napi_gro_receive
    			napi_skb_finish
    				netif_receive_skb_internal
    	//申请新skb，更新desc，kick后端以供后端收包
        try_fill_recv
            do {
                //
                add_recvbuf_mergeable
                //mtu > 1500表示大包
                add_recvbuf_big
                add_recvbuf_small
            } while (rq->vq->num_free);
            //前端循环一次会填充Buffer到avail ring
            virtqueue_kick(rq->vq);
				virtqueue_kick_prepare
                    virtqueue_notify
                    	vp_notify
                    		//把vq的index编号写入到设备的IO地址空间中
                    		//实际上就是设备对应的PCI配置空间中VIRTIO_PCI_QUEUE_NOTIFY位置。这里执行IO操作会引发VM-exit，，继而退出到KVM->qemu中处理
                    		iowrite16(vq->index, (void __iomem *)vq->priv);
```



```c
virtqueue_kick_prepare
    //event配置为VIRTIO_RING_F_EVENT_IDX
    //old表示add之前的avail idx
    //new表示add之后的avail idx
    //vring_avail_event(&vq->vring)表示used ring的最后一个元素的值
	vring_need_event(virtio16_to_cpu(_vq->vdev, vring_avail_event(&vq->vring)), new, old);
			
#define vring_avail_event(vr) (*(__virtio16 *)&(vr)->used->ring[(vr)->num])

//返回true表示后端处理快，可以kick后端；
//[old---event---new]
//返回false表示后端当前处理位置落后于old，速度慢，等下次一起kick后端
//[event---old---new]
static inline int vring_need_event(__u16 event_idx, __u16 new_idx, __u16 old)
{
	/* Note: Xen has similar logic for notification hold-off
	 * in include/xen/interface/io/ring.h with req_event and req_prod
	 * corresponding to event_idx + 1 and new_idx respectively.
	 * Note also that req_event and req_prod in Xen start at 1,
	 * event indexes in virtio start at 0. */
	return (__u16)(new_idx - event_idx - 1) < (__u16)(new_idx - old);
}
```



② used_ring中的avail_event

a. 由后端驱动（Host）设置，标识希望前端驱动（Guest）触发中断的阈值

b. 前端驱动（Guest）在向Avail Ring加入buffer后，检查Avail Ring的idx字段，只有达到阈值才触发中断

后端逻辑（**vhost_need_event**），详见以下链接，不展开

http://blog.chinaunix.net/uid-28541347-id-5783785.html



vring结构总结如下图：

![0](https://img-blog.csdnimg.cn/img_convert/e685378888da6bc6cda4993cb7098449.png)



### 6、vring_virtqueue结构

用于描述前端驱动guest的一条虚拟队列

```c
struct vring_virtqueue {
	struct virtqueue vq;

	/* Actual memory layout for this queue */
	struct vring vring;

	/* Can we use weak barriers? */
	bool weak_barriers;

	/* Other side has made a mess, don't try any more. */
	bool broken;

	/* Host supports indirect buffers */
	bool indirect;

	/* Host publishes avail event idx */
	bool event;

	/* Head of free buffer list. */
	unsigned int free_head;
	/* Number we've added since last sync. */
	unsigned int num_added; //上次通知后端驱动后avail ring增加的数目

	/* Last used index we've seen. */
	u16 last_used_idx;

	/* Last written value to avail->flags */
	u16 avail_flags_shadow;

	/* Last written value to avail->idx in guest byte order */
	u16 avail_idx_shadow;

	/* How to notify other side. FIXME: commonalize hcalls! */
	bool (*notify)(struct virtqueue *vq);

	/* DMA, allocation, and size information */
	bool we_own_ring;
	size_t queue_size_in_bytes;
	dma_addr_t queue_dma_addr;

#ifdef DEBUG
	/* They're supposed to lock for us. */
	unsigned int in_use;

	/* Figure out if their kicks are too delayed. */
	bool last_add_time_valid;
	ktime_t last_add_time;
#endif

	/* Per-descriptor state. */
	struct vring_desc_state desc_state[];
};
```



# 三、virtio操作

## 1、创建virtqueue

```c
//在virtio框架中，首先向虚拟机注册的是一个pci_driver
//后续所有vitio设备均是以虚拟pci设备的形式注册
virtio_pci_driver
	.probe = virtio_pci_probe
    	//cat /sys/module/virtio_pci/parameters/force_legacy
    	//force_legacy为false
    		virtio_pci_modern_probe
    			//设置virtio_config_ops回调
    			vp_dev->vdev.config = &virtio_pci_config_ops;
					vp_modern_find_vqs
                //设置setup_vq回调
				vp_dev->setup_vq = setup_vq;
		//注册virtio设备
		register_virtio_device
            dev->dev.bus = &virtio_bus;

//注册bus_type
virtio_dev_probe //virtio_bus
    drv->probe(dev);
		virtnet_probe //virtio_driver
			
virtnet_probe
	err = init_vqs(vi);
		ret = virtnet_find_vqs(vi);
			callbacks[rxq2vq(i)] = skb_recv_done;
			callbacks[txq2vq(i)] = skb_xmit_done;
			vi->vdev->config->find_vqs(vi->vdev, total_vqs, vqs, callbacks, names, ctx, NULL)
				vp_modern_find_vqs
					vp_find_vqs
						vp_find_vqs_msix
                			//为rx,tx各创建virtqueue
							vp_setup_vq
								vq = vp_dev->setup_vq(vp_dev, info, index, callback, name, ctx, msix_vec);
									//设置了notify hook函数，用于前端触发后端中断
									vring_create_virtqueue(index, num, SMP_CACHE_BYTES, &vp_dev->vdev, true, true, ctx, vp_notify, callback, name);
                                        //分配连续物理内存
                                        vring_create_virtqueue
                                        //生成vring_virtqueue结构并初始化
                                        __vring_new_virtqueue
```



## 2、前端驱动发送数据

### （1）流程

step 1：从descriptor table中取出描述符

step 2：根据要发送的数据填充并组织描述符

step 3：将组织好的描述符加入avail ring

step 4：触发后端驱动中断，通知其处理数据

### （2）virtqueue_add函数

virtqueue_add函数封装了一次数据请求，所有out & in请求均组织为一个decriptor chain提交到avail ring



virtqueue_add函数被封装为如下4种方式供前端驱动调用，

① virtqueue_add_sgs

可以同时提交out & in数据请求，且个数可设置

```c
int virtqueue_add_sgs(struct virtqueue *_vq,
		      struct scatterlist *sgs[],
		      unsigned int out_sgs,
		      unsigned int in_sgs,
		      void *data,
		      gfp_t gfp)
{
	unsigned int i, total_sg = 0;

	/* Count them first. */
	for (i = 0; i < out_sgs + in_sgs; i++) {
		struct scatterlist *sg;
        //out&in数据累加
		for (sg = sgs[i]; sg; sg = sg_next(sg))
			total_sg++;
	}
	return virtqueue_add(_vq, sgs, total_sg, out_sgs, in_sgs,
			     data, NULL, gfp);
}
```

② virtqueue_add_outbuf

只提交一个out数据请求

```c
int virtqueue_add_outbuf(struct virtqueue *vq,
			 struct scatterlist *sg, unsigned int num,
			 void *data,
			 gfp_t gfp)
{
	return virtqueue_add(vq, &sg, num, 1, 0, data, NULL, gfp);
}
```

③ virtqueue_add_inbuf

只提交一个in数据请求

```c
int virtqueue_add_inbuf(struct virtqueue *vq,
			struct scatterlist *sg, unsigned int num,
			void *data,
			gfp_t gfp)
{
	return virtqueue_add(vq, &sg, num, 0, 1, data, NULL, gfp);
}
```

④ virtqueue_add_inbuf_ctx

只提交一个in数据请求，且携带上下文信息ctx

```c
int virtqueue_add_inbuf_ctx(struct virtqueue *vq,
			struct scatterlist *sg, unsigned int num,
			void *data,
			void *ctx,
			gfp_t gfp)
{
	return virtqueue_add(vq, &sg, num, 0, 1, data, ctx, gfp);
}
```



## 3、前端触发中断

```c
virtqueue_kick
	virqueue_kick_prepare
		virtqueue_notify
			//通知后端 只return true
			vp_notify 
```



```c
bool virtqueue_kick_prepare(struct virtqueue *_vq)
{
	struct vring_virtqueue *vq = to_vvq(_vq);
	u16 new, old;
	bool needs_kick;

	START_USE(vq);
	/* We need to expose available array entries before checking avail
	 * event. */
	virtio_mb(vq->weak_barriers);
	
   	//上一次avail ring的idx
	old = vq->avail_idx_shadow - vq->num_added;
    //最后一次avail ring的idx
	new = vq->avail_idx_shadow;
	vq->num_added = 0;

#ifdef DEBUG
	if (vq->last_add_time_valid) {
		WARN_ON(ktime_to_ms(ktime_sub(ktime_get(),
					      vq->last_add_time)) > 100);
	}
	vq->last_add_time_valid = false;
#endif

	if (vq->event) {
		needs_kick = vring_need_event(virtio16_to_cpu(_vq->vdev, vring_avail_event(&vq->vring)),
					      new, old);
	} else {
		needs_kick = !(vq->vring.used->flags & cpu_to_virtio16(_vq->vdev, VRING_USED_F_NO_NOTIFY));
	}
	END_USE(vq);
	return needs_kick;
}
```



## 4、前端被触发中断

#### （1）创建virtqueue时，会为每条virtqueue注册中断

```c
virtnet_probe
	err = init_vqs(vi);
		ret = virtnet_find_vqs(vi);
			callbacks[rxq2vq(i)] = skb_recv_done;
			callbacks[txq2vq(i)] = skb_xmit_done;
			vi->vdev->config->find_vqs(vi->vdev, total_vqs, vqs, callbacks, names, ctx, NULL)
				vp_modern_find_vqs
					vp_find_vqs
                		vp_find_vqs_msix
                			request_irq(pci_irq_vector(vp_dev->pci_dev, msix_vec), vring_interrupt, 0, vp_dev->msix_names[msix_vec], vqs[i]);
```

中断处理函数为vring_interrupt

```c
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)
{
	return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}

int request_threaded_irq(unsigned int irq, irq_handler_t handler,
			 irq_handler_t thread_fn, unsigned long irqflags,
			 const char *devname, void *dev_id)
{
    struct irqaction *action;
    ...
    action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
    action->handler = handler; //vring_interrupt
    ...
}
```

#### （2）vring_interrupt函数

vring_interrupt函数的核心操作是调用创建virtqueue时注册的callback回调函数，即：

skb_recv_done //接受

skb_xmit_done //发送

```c
irqreturn_t vring_interrupt(int irq, void *_vq)
{
	struct vring_virtqueue *vq = to_vvq(_vq);

	if (!more_used(vq)) {
		pr_debug("virtqueue interrupt with no work for %p\n", vq);
		return IRQ_NONE;
	}

	if (unlikely(vq->broken))
		return IRQ_HANDLED;

	pr_debug("virtqueue callback for %p (%p)\n", vq, vq->vq.callback);
	if (vq->vq.callback)
		vq->vq.callback(&vq->vq);

	return IRQ_HANDLED;
}
EXPORT_SYMBOL_GPL(vring_interrupt);
```



## 5、前端接受数据

### 大致流程

```c
//每个vq 对应一个数据接收函数 vring_interrupt()
vring_interrupt(int irq, void *_vq)
    skb_recv_done(struct virtqueue *rvq) //virtio_net.c 中 virtnet_find_vqs() 中，数据接收完成回调函数
        virtqueue_napi_schedule(struct napi_struct *napi, struct virtqueue *vq)
    		//关闭中断，vhost不会通知前端
            virtqueue_disable_cb(vq);
            //检查poll队列上是否有设备在等待轮询
			__napi_schedule
                ____napi_schedule(this_cpu_ptr(&softnet_data), n);
					//把 NAPI 加入到本地cpu的 softnet_data 的 poll_list链表头
					list_add_tail(&napi->poll_list, &sd->poll_list);
					//调度收包软中断
					__raise_softirq_irqoff(NET_RX_SOFTIRQ);
						net_rx_action //net/core/dev.c中的net_dev_init()中，软中断回调函数
							//ksoftirqd最多一次处理300个包（sysctl -a | grep netdev_budget）
                            budget -= napi_poll(n, &repoll);
								//测试已设置NAPI_STATE_SCHED，表示NAPI有报文需要接受
								//drivers/net/virtio_net.c中的virtnet_alloc_queues中，为virtnet_poll设置了NAPI_STATE_SCHED
								//work表示收包数
								work = n->poll(n, weight);

virtnet_poll(struct napi_struct *napi, int budget)
	received = virtnet_receive(rq, budget, &xdp_xmit);
```



### virtnet_receive函数分析：

```c
static int virtnet_receive(struct receive_queue *rq, int budget,
			   unsigned int *xdp_xmit)
{
	struct virtnet_info *vi = rq->vq->vdev->priv;
	unsigned int len, received = 0, bytes = 0;
	void *buf;
	
    //非大包 或者 host对于大包进行合并rx buffers
	if (!vi->big_packets || vi->mergeable_rx_bufs) {
		void *ctx;

		while (received < budget &&
		       (buf = virtqueue_get_buf_ctx(rq->vq, &len, &ctx))) {
			receive_buf(vi, rq, buf, len, ctx, xdp_xmit, &bytes);
			received++;
		}
	} else {
		while (received < budget &&
		       (buf = virtqueue_get_buf(rq->vq, &len)) != NULL) {
			receive_buf(vi, rq, buf, len, NULL, xdp_xmit, &bytes);
			received++;
		}
	}

	if (rq->vq->num_free > virtqueue_get_vring_size(rq->vq) / 2) {
		if (!try_fill_recv(vi, rq, GFP_ATOMIC))
			schedule_delayed_work(&vi->refill, 0);
	}

	u64_stats_update_begin(&rq->stats.syncp);
	rq->stats.bytes += bytes;
	rq->stats.packets += received;
	u64_stats_update_end(&rq->stats.syncp);

	return received;
}
```

#### （1）mergeable_rx_buf设置

依赖于vhost是否有VIRTIO_NET_F_MRG_RXBUF的feature

```c
virtnet_probe
    #define VIRTIO_NET_F_MRG_RXBUF	15	/* Host can merge receive buffers. */
        if (virtio_has_feature(vdev, VIRTIO_NET_F_MRG_RXBUF))
            vi->mergeable_rx_bufs = true;
        ...
```

#### （2）big_packets设置



```c
virtnet_probe
    #define VIRTIO_NET_F_GUEST_TSO4	7	/* Guest can handle TSOv4 in. */
    #define VIRTIO_NET_F_GUEST_TSO6	8	/* Guest can handle TSOv6 in. */
    #define VIRTIO_NET_F_GUEST_ECN	9	/* Guest can handle TSO[6] w/ ECN in. */
    #define VIRTIO_NET_F_GUEST_UFO	10	/* Guest can handle UFO in. */
    /* If we can receive ANY GSO packets, we must allocate large ones. */
	if (virtio_has_feature(vdev, VIRTIO_NET_F_GUEST_TSO4) ||
	    virtio_has_feature(vdev, VIRTIO_NET_F_GUEST_TSO6) ||
	    virtio_has_feature(vdev, VIRTIO_NET_F_GUEST_ECN) ||
	    virtio_has_feature(vdev, VIRTIO_NET_F_GUEST_UFO))
		vi->big_packets = true;
        
    if (virtio_has_feature(vdev, VIRTIO_NET_F_MTU)) {
        ...
        if (dev->mtu > ETH_DATA_LEN)
			vi->big_packets = true;
    }
```







```c
void *virtqueue_get_buf_ctx(struct virtqueue *_vq, unsigned int *len,
			    void **ctx)
{
	struct vring_virtqueue *vq = to_vvq(_vq);
	void *ret;
	unsigned int i;
	u16 last_used;

	START_USE(vq);

	if (unlikely(vq->broken)) {
		END_USE(vq);
		return NULL;
	}

	if (!more_used(vq)) {
		pr_debug("No more buffers in queue\n");
		END_USE(vq);
		return NULL;
	}

	/* Only get used array entries after they have been exposed by host. */
	virtio_rmb(vq->weak_barriers);

	last_used = (vq->last_used_idx & (vq->vring.num - 1));
	i = virtio32_to_cpu(_vq->vdev, vq->vring.used->ring[last_used].id);
	*len = virtio32_to_cpu(_vq->vdev, vq->vring.used->ring[last_used].len);

	if (unlikely(i >= vq->vring.num)) {
		BAD_RING(vq, "id %u out of range\n", i);
		return NULL;
	}
	if (unlikely(!vq->desc_state[i].data)) {
		BAD_RING(vq, "id %u is not a head!\n", i);
		return NULL;
	}

	/* detach_buf clears data, so grab it now. */
	ret = vq->desc_state[i].data;
	detach_buf(vq, i, ctx);
	vq->last_used_idx++;
	/* If we expect an interrupt for the next entry, tell host
	 * by writing event index and flush out the write before
	 * the read in the next get_buf call. */
	if (!(vq->avail_flags_shadow & VRING_AVAIL_F_NO_INTERRUPT))
		virtio_store_mb(vq->weak_barriers,
				&vring_used_event(&vq->vring),
				cpu_to_virtio16(_vq->vdev, vq->last_used_idx));

#ifdef DEBUG
	vq->last_add_time_valid = false;
#endif

	END_USE(vq);
	return ret;
}
```



## 收包流程

```c
数据接收流程：
 
 napi_gro_receive(&rq->napi, skb);
                netif_receive_skb
                    __netif_receive_skb         // 传输skb给网络层
    /\
    ||
驱动 virtio_net.c 中poll方法 napi_poll(n, &repoll); 即virtio_net.c 中 virtnet_poll()
 virtnet_poll
     virtnet_receive
         receive_buf                 // 接收到的数据转换成skb
             //根据接收类型XDP_PASS、XDP_TX等对 virtqueue 中的数据进行不同的处理
             skb = receive_mergeable(dev, vi, rq, buf, ctx, len, xdp_xmit,stats); or
             skb = receive_big(dev, vi, rq, buf, len, stats);  or
             skb = receive_small(dev, vi, rq, buf, ctx, len, xdp_xmit, stats);              
             napi_gro_receive(&rq->napi, skb);   // 把skb上传到上层协议栈
                 
                     
         schedule_delayed_work                  //通过你延迟队列接收数据
             refill_work
                 try_fill_recv(vi, rq, GFP_KERNEL);
                 如果检测到本次中断 receive 数据完成，则重新开启中断                                
                local_bh_enable                 //enable 软中断 等待下一次中断接收数据
	virtqueue_napi_complete //开中断
    /\
    ||
中断下半步
 
执行软中断回调函数 net_rx_action(), 调用 virtio_net.c 中 virtnet_poll()
 
    /\
    ||
检查poll队列上是否有设备在等待轮询
napi_schedule ->__napi_schedule  ->   list_add_tail(&napi->poll_list, &sd->poll_list); //把 NAPI 加入到本地cpu的 softnet_data 的 poll_list链表头
        __raise_softirq_irqoff(NET_RX_SOFTIRQ);          // 调度收包软中断
                     
    /\
    ||
skb_recv_done           //virtio_net.c 中 virtnet_find_vqs() 中，数据接收完成回调函数
    virtqueue_napi_schedule
        virtqueue_disable_cb(vq);
        napi_schedule
            __napi_schedule
    /\
    ||
每个vq 对应一个数据接收函数 vring_interrupt()
vring_interrupt()       //virtio_ring.c
    vq->vq.callback(&vq->vq);   即virtio_net.c 中 skb_recv_done
		skb_recv_done           //硬中断回调函数
    		virtqueue_napi_schedule
                virtqueue_disable_cb(vq); //关中断进入NAPI模式
				__napi_schedule(napi);
					__raise_softirq_irqoff(NET_RX_SOFTIRQ); //调度收包软中断
    /\
    ||
中断上半步
pcie网卡发送数据给host时，会触发pci msix硬中断，然后host driver agile_nic.c 中执行回调函数vring_interrupt
```



# 参考链接

1、【原创】Linux虚拟化KVM-Qemu分析（十一）之virtqueue

https://www.cnblogs.com/LoyenWang/p/14589296.html

2、VirtIO实现原理——PCI基础

https://blog.csdn.net/huang987246510/article/details/103379926

3、virtio前后端配合限速分析

http://blog.chinaunix.net/uid-28541347-id-5783785.html

4、virtIO前后端notify机制详解（<b style="color:red;">对于前后端使用virtqueue图解很清晰！</b>）

https://www.sohu.com/a/239256171_467784

5、virtio_net.c 驱动中数据收发流程

https://blog.csdn.net/weixin_40209911/article/details/119491380

6、深入理解Linux网络之网络性能优化建议

https://blog.csdn.net/QTM_Gitee/article/details/125229447
