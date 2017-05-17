---
layout: post
title: kvm block IO 之一：qemu block drive初始化过程分析
category: KVM虚拟化
tags: [KVM，qemu]
keywords: KVM，qemu
description: 
---

> 用正确的工具，做正确的事情

block drive初始化的过程主要完成相关结构变量的创建，结构变量成员的赋值以及队列的构造等等，block drive初始化过程创建的结构变量以及结构变量之间的主要关系，如下图所示。

![block driver 初始化相关结构变量](http://7u2rbh.com1.z0.glb.clouddn.com/block-driver-init.png)
	
上图只标出了结构变量以及队列结构之间的主要关系，下面重点总结相关的数据结构定义以及结构变量的创建过程。

## 相关数据结构定义

block\_backends、all\_bdrv\_states、bdrv\_drivers都为全局static结构变量，他们在block.c文件中被定义，如下：

	static QTAILQ_HEAD(, BlockDriverState) all_bdrv_states =
    	QTAILQ_HEAD_INITIALIZER(all_bdrv_states);
	
	static QLIST_HEAD(, BlockDriver) bdrv_drivers =
    	QLIST_HEAD_INITIALIZER(bdrv_drivers);

	static QTAILQ_HEAD(, BlockBackend) block_backends =
    QTAILQ_HEAD_INITIALIZER(block_backends);

block\_backends、all\_bdrv\_states、bdrv\_drivers均为队列类型的变量，其中队列的元素类型为分别BlockBackend、BlockDriver以及BlockDriverState，这三个类型在block-backend.c和block\_int.h中定义，其中BlockBackend详细定义如下：

	struct BlockBackend {
    	char *name;
    	int refcnt;
    	BdrvChild *root;
    	DriveInfo *legacy_dinfo;    /* null unless created by drive_new() */
    	QTAILQ_ENTRY(BlockBackend) link;         /* for block_backends */
    	QTAILQ_ENTRY(BlockBackend) monitor_link; /* for monitor_block_backends */
    	BlockBackendPublic public;
	
    	void *dev;                  /* attached device model, if any */
    	bool legacy_dev;            /* true if dev is not a DeviceState */
    	/* TODO change to DeviceState when all users are qdevified */
    	const BlockDevOps *dev_ops;
    	void *dev_opaque;
	
    	/* the block size for which the guest device expects atomicity */
    	int guest_block_size;
	
    	/* If the BDS tree is removed, some of its options are stored here (which
    	 * can be used to restore those options in the new BDS on insert) */
    	BlockBackendRootState root_state;
	
    	bool enable_write_cache;
	
    	/* I/O stats (display with "info blockstats"). */
    	BlockAcctStats stats;
	
    	BlockdevOnError on_read_error, on_write_error;
    	bool iostatus_enabled;
    	BlockDeviceIoStatus iostatus;
	
    	bool allow_write_beyond_eof;
	
    	NotifierList remove_bs_notifiers, insert_bs_notifiers;
	};	


BlockDriver的详细定义如下：

	struct BlockDriver {
    	const char *format_name;
    	int instance_size;
	
    	/* set to true if the BlockDriver is a block filter */
    	bool is_filter;
    	/* for snapshots block filter like Quorum can implement the
    	 * following recursive callback.
    	 * It's purpose is to recurse on the filter children while calling
    	 * bdrv_recurse_is_first_non_filter on them.
    	 * For a sample implementation look in the future Quorum block filter.
    	 */
    	bool (*bdrv_recurse_is_first_non_filter)(BlockDriverState *bs,
    	                                         BlockDriverState *candidate);
	
    	int (*bdrv_probe)(const uint8_t *buf, int buf_size, const char *filename);
    	int (*bdrv_probe_device)(const char *filename);
	
    	/* Any driver implementing this callback is expected to be able to handle
    	 * NULL file names in its .bdrv_open() implementation */
    	void (*bdrv_parse_filename)(const char *filename, QDict *options, Error **errp);
    	/* Drivers not implementing bdrv_parse_filename nor bdrv_open should have
    	 * this field set to true, except ones that are defined only by their
    	 * child's bs.
    	 * An example of the last type will be the quorum block driver.
    	 */
    	bool bdrv_needs_filename;
	
    	/* Set if a driver can support backing files */
    	bool supports_backing;
	
    	/* For handling image reopen for split or non-split files */
    	int (*bdrv_reopen_prepare)(BDRVReopenState *reopen_state,
    	                           BlockReopenQueue *queue, Error **errp);
    	void (*bdrv_reopen_commit)(BDRVReopenState *reopen_state);
    	void (*bdrv_reopen_abort)(BDRVReopenState *reopen_state);
    	void (*bdrv_join_options)(QDict *options, QDict *old_options);
		............
		}

BlockDriver主要由函数类型的成员组成，包括了后端块设备支持的所有的方法，比较重要的非函数类型的成员的format\_name为该块驱动所属镜像类型的名称。BlockDriverState的定义如下：

	struct BlockDriverState {
    	int64_t total_sectors; /* if we are reading a disk image, give its
    	                          size in sectors */
    	int open_flags; /* flags used to open the file, re-used for re-open */
    	bool read_only; /* if true, the media is read only */
    	bool encrypted; /* if true, the media is encrypted */
    	bool valid_key; /* if true, a valid encryption key has been set */
    	bool sg;        /* if true, the device is a /dev/sg* */
    	bool probed;    /* if true, format was probed rather than specified */
	
    	int copy_on_read; /* if nonzero, copy read backing sectors into image.
    	                     note this is a reference count */
	
    	CoQueue flush_queue;            /* Serializing flush queue */
    	bool active_flush_req;          /* Flush request in flight? */
    	unsigned int write_gen;         /* Current data generation */
    	unsigned int flushed_gen;       /* Flushed write generation */
	
    	BlockDriver *drv; /* NULL means no media */
    	void *opaque;
	
    	AioContext *aio_context; /* event loop used for fd handlers, timers, etc */
    	/* long-running tasks intended to always use the same AioContext as this
    	 * BDS may register themselves in this list to be notified of changes
    	 * regarding this BDS's context */
    	QLIST_HEAD(, BdrvAioNotifier) aio_notifiers;
    	bool walking_aio_notifiers; /* to make removal during iteration safe */
	
    	char filename[PATH_MAX];
    	char backing_file[PATH_MAX]; /* if non zero, the image is a diff of
    	                                this file image */
    	char backing_format[16]; /* if non-zero and backing_file exists */
	
		..........................
    	bool wakeup;
	
    	/* Offset after the highest byte written to */
    	uint64_t wr_highest_offset;
	
    	/* I/O Limits */
    	BlockLimits bl;
	
    	/* Flags honored during pwrite (so far: BDRV_REQ_FUA) */
    	unsigned int supported_write_flags;
    	/* Flags honored during pwrite_zeroes (so far: BDRV_REQ_FUA,
    	 * BDRV_REQ_MAY_UNMAP) */
    	unsigned int supported_zero_flags;
	
    	/* the following member gives a name to every node on the bs graph. */
    	char node_name[32];

	    ..........................
    	/* threshold limit for writes, in bytes. "High water mark". */
    	uint64_t write_threshold_offset;
    	NotifierWithReturn write_threshold_notifier;
	
    	/* counters for nested bdrv_io_plug and bdrv_io_unplugged_begin */
    	unsigned io_plugged;
    	unsigned io_plug_disabled;
	
    	int quiesce_counter;
	}	

BlockDriverState主要由非函数类型的成员组成，比较重要的成员是BlockDriver *drv， 该成员表明BlockDriverState所述的块设备驱动。总体来说，BlockDriver定义了块设备支持的功能方法，BlockDriverState定义描述了块设备本身的状态信息。
	
## 结构变量的创建及初始化

上述bdrv\_drivers与all\_bdrv\_states为全局静态队列类型的变量，在block.c文件中被创建并初始化为空队列，但是真实队列的生成是在qemu启动过程初始化阶段来完成的。

### bdrv\_drivers队列的建立

对于raw格式镜像来说其相关的bdrv\_drivers 对象bdrv\_raw在raw\_bsd.c被创建， 为全局类型的变量。bdrv\_raw被插入到bdrv\_drivers函数调用流程如下：

	(gdb) bt
	#0  bdrv_register (bdrv=0x7fd9772423c0 <bdrv_raw>) at block.c:229
	#1  0x00007fd9767d6870 in bdrv_raw_init () at block/raw_bsd.c:487
	#2  0x00007fd97689fbf0 in module_call_init (type=MODULE_INIT_BLOCK) at util/module.c:101
	#3  0x00007fd9767ca570 in bdrv_init () at block.c:3234
	#4  0x00007fd9767ca585 in bdrv_init_with_whitelist () at block.c:3240
	#5  0x00007fd976585e58 in main (argc=12, argv=0x7ffc4feb0288, envp=0x7ffc4feb02f0) at vl.c:3101

除了从module\_call\_init到bdrv\_raw\_init这一调用关系只玩，其他的均为简单的函数直接调用关系，module\_call\_init到bdrv\_raw\_init经过了回调函数， 接下来重点分析一下module\_call\_init的实现以及相关的数据结构， module\_call\_init的详细实现如下：

	 91 void module_call_init(module_init_type type)
	 92 {
	 93
	 94     ModuleTypeList *l;
	 95     ModuleEntry *e;
	 96
	 97     l = find_type(type);
	 98
	 99     QTAILQ_FOREACH(e, l, node) {
	100         e->init();
	101     }
	102 }

module\_call\_init的入参是一个枚举类型的常量，用于表示要初始化的模块的类型，这里常量的值为MODULE\_INIT\_BLOCK。94~95行创建了两个指针变量l、e， 这里有两个新的类型ModuleTypeList和ModuleEntry，其中ModuleTypeList为一个尾队列类型的结构，该队列的元素是ModuleEntry类型，详细定义如下：

	typedef struct ModuleEntry
	{
    	void (*init)(void);
    	QTAILQ_ENTRY(ModuleEntry) node;
    	module_init_type type;
	} ModuleEntry;

该结构包括3个成员，第一个是成员是回调函数类型，第二个成员是尾队列的结构指针，第三个成员是枚举常量，表示相关模块的类型，继续回到函数module\_call\_init实现分析上来。97行通过直接调用find\_type函数得到一个ModuleTypeList的地址并赋值给l，我们知道l是一个尾队列类型的指针，队列的元素为ModuleEntry， 那么下面的99~101就比较好理解，只是遍历通过find\_type得到的尾队列的每一个ModuleEntry元素，并调用该元素的回调函数init。上面bdrv\_drivers的建立过程，对于bdrv\_raw被插入到bdrv\_drivers的过程，就是通过该init回调函数来调用bdrv\_raw\_init来实现的,那么这里有一个问题,init是怎么被注册的的？？？为此需要进一步分析如果通过find\_type函数的实现以及相关的数据结构,

	33 static ModuleTypeList init_type_list[MODULE_INIT_MAX];
	34
	35 static ModuleTypeList dso_init_list;
	36
	37 static void init_lists(void)
	38 {
	39     static int inited;
	40     int i;
	41
	42     if (inited) {
	43         return;
	44     }
	45
	46     for (i = 0; i < MODULE_INIT_MAX; i++) {
	47         QTAILQ_INIT(&init_type_list[i]);
	48     }
	49
	50     QTAILQ_INIT(&dso_init_list);
	51
	52     inited = 1;
	53 }
	54
	55
	56 static ModuleTypeList *find_type(module_init_type type)
	57 {
	58     init_lists();
	59
	60     return &init_type_list[type];
	61 }

33行，qemu定义了一个全局静态ModuleTypeList类型的数组变量init\_type\_list，数组的下标即为该ModuleTypeList元素相关的模块类型，37~53行init\_list函数用于初始化init\_type\_list变量，该函数定义了一个局部静态变量inited，用于表明该数组是否已经被初始化过了，如果已经被初始化则直接返回，否则初始化每个数组元素。find\_type函数，首先调用init\_lists，然后直接通过枚举常量type作为下标在静态全局数字中获取到一个ModuleTypeList对象即可。分析回调函数init是在什么地方注册的，关键是全局静态变量init\_type\_list中的每个元素，也即每个尾队列是如果被创建的，module.c文件register\_module\_init()函数提供了对init\_type\_list中每个尾队列插入元素的操作，详细实现如下：

	63 void register_module_init(void (*fn)(void), module_init_type type)
	64 {
	66     ModuleEntry *e;
	67     ModuleTypeList *l;
	68
	69     e = g_malloc0(sizeof(*e));
	70     e->init = fn;
	71     e->type = type;
	72
	73     l = find_type(type);
	74
	75     QTAILQ_INSERT_TAIL(l, e, node);
	76 }
	

该函数的实现比较简单，创建元素并根据入参进行初始化，然后将元素插入对应的尾队列，问题的关键是该函数如何被调用执行，我们通过gdb跟踪代码得到该函数的调用堆栈信息如下：

	(gdb) bt
	#0  register_module_init (fn=0x7f69ed44b1b9 <bdrv_qcow_init>, type=MODULE_INIT_BLOCK) at util/module.c:72
	#1  0x00007f69ed44b1e0 in do_qemu_init_bdrv_qcow_init () at block/qcow.c:1065
	#2  0x00007f69ed8a0bfd in __libc_csu_init ()
	#3  0x00007f69eb10dac5 in __libc_start_main () from /usr/lib64/libc.so.6
	#4  0x00007f69ed06a9d9 in _start ()

从调用堆栈信息中可以看到register\_module\_init函数是在qemu main函数执行之前被调用执行的，并且在block/qcow.c代码源文件中并没有找到do\_qemu\_init\_bdrv\_qcow\_init函数的定义，在main函数之前执行一些函数代码，这个是如何实现的呢？详细分析见<qemu中的module机制>。总结一下，bdrv\_drivers队列的建立过程如下：

> qemu main函数之前通过register\_module\_init函数将raw block 模块初始化函数bdrv\_raw\_init的注册到ModuleEntry对象的init成员，并将该对象保存到到静态全局变量init\_type\_list中，这里涉及到qemu module机制。在qemu main函数执行过程中，通过module\_call\_init遍历bdrv\_raw\_init中相关ModuleTypeList队列中每个元素并执行回调函数init，进入调用bdrv_raw\_init（）函数将全局变量bdrv\_raw添加到bdrv\_drivers队列中。

这里比较有意思的是qemu实现过程中的module机制，这个将再单独进行分析和总结。

### block\_backends和all\_bdrv\_states队列的建立

对于raw block driver初始化过程， blk\_new\_open（）函数通过直接调用blk\_new（）函数创建blk，函数blk\_new（）创建blk同时将blk插入到block\_backends队列中，同时函数blk\_new\_open（）通过直接调用bdrv\_open（），该函数返回一个BlockDriverState结构变量。bdrv\_open（）经过一系列的函数调用最终在bdrv\_open\_inherit（）函数中，通过调用函数bdrv\_new（）来创建一个bs，同时将该bs插入到全局队列all\_bdrv\_states中，因此：

> bs是在blk的创建过程中同时创建的，bs和blk都是通过专门的结构变量创建函数来创建，并且在创建后相关的结构变量被直接插入到相关的全局队列中，已被之后查找使用。

进入bdrv\_open\_inherit函数的调用堆栈如下：

	(gdb) bt
	#0  bdrv_open_inherit (
    	filename=0x7f48189e8e50 "/home/vms/centos_6_8_ldd.img",
    	reference=0x0, options=0x7f48189e7d90, flags=128, parent=0x0,
    	child_role=0x0, errp=0x7ffee15a9678) at block.c:1712
	#1  0x00007f48178c7b6d in bdrv_open (
    	filename=0x7f48189e8e50 "/home/vms/centos_6_8_ldd.img",
    	reference=0x0, options=0x7f48189e7d90, flags=128, errp=0x7ffee15a9678)
    	at block.c:1941
	#2  0x00007f4817918167 in blk_new_open (
    	filename=0x7f48189e8e50 "/home/vms/centos_6_8_ldd.img",
    	reference=0x0, options=0x7f48189e7d90, flags=128, errp=0x7ffee15a9678)
    	at block/block-backend.c:161
	#3  0x00007f4817667b53 in blockdev_init (
    	file=0x7f48189e8e50 "/home/vms/centos_6_8_ldd.img",
    	bs_opts=0x7f48189e7d90, errp=0x7ffee15a9678) at blockdev.c:585
	#4  0x00007f4817668ca1 in drive_new (all_opts=0x7f481896ac60,
    	block_default_type=IF_IDE) at blockdev.c:1084
	#5  0x00007f4817680cb8 in drive_init_func (opaque=0x7f4818988e80,
    	opts=0x7f481896ac60, errp=0x0) at vl.c:1188
	#6  0x00007f48179b1ea6 in qemu_opts_foreach (
    	list=0x7f481821f2a0 <qemu_drive_opts>,
    	func=0x7f4817680c7c <drive_init_func>, opaque=0x7f4818988e80,
    	errp=0x0) at util/qemu-option.c:1120
	#7  0x00007f4817689356 in main (argc=51, argv=0x7ffee15a9b88,
    	envp=0x7ffee15a9d28) at vl.c:4484

下面详细分析bdrv\_open\_inherit（）函数的实现：

	1700 static BlockDriverState *bdrv_open_inherit(const char *filename,
	1701                                            const char *reference,
	1702                                            QDict *options, int flags,
	1703                                            BlockDriverState *parent,
	1704                                            const BdrvChildRole *child_role,
	1705                                            Error **errp)
	1706 {
	1707     int ret;
	1708     BdrvChild *file = NULL;
	1709     BlockDriverState *bs;
	1710     BlockDriver *drv = NULL;
	1711     const char *drvname;
	1712     const char *backing;
	1713     Error *local_err = NULL;
	1714     QDict *snapshot_options = NULL;
	1715     int snapshot_flags = 0;
	......
	1720     if (reference) {
	1721         bool options_non_empty = options ? qdict_size(options) : false;
	1722         QDECREF(options);
	1723
	1724         if (filename || options_non_empty) {
	1725             error_setg(errp, "Cannot reference an existing block device with "
	1726                        "additional options or a new filename");
	1727             return NULL;
	1728         }
	1729
	1730         bs = bdrv_lookup_bs(reference, reference, errp);
	1731         if (!bs) {
	1732             return NULL;
	1733         }
	1734
	1735         bdrv_ref(bs);
	1736         return bs;
	1737     }
	1738
	1739     bs = bdrv_new();
	......
	1752     bs->explicit_options = qdict_clone_shallow(options);
	1753
	1754     if (child_role) {
	1755         bs->inherits_from = parent;
	1756         child_role->inherit_options(&flags, options,
	1757                                     parent->open_flags, parent->options);
	1758     }
	......
	1788     /* Find the right image format driver */
	1789     drvname = qdict_get_try_str(options, "driver");
	1790     if (drvname) {
	1791         drv = bdrv_find_format(drvname);
	1792         if (!drv) {
	1793             error_setg(errp, "Unknown driver: '%s'", drvname);
	1794             goto fail;
	1795         }
	1796     }
	......
	1806     /* Open image file without format layer */
	1807     if ((flags & BDRV_O_PROTOCOL) == 0) {
	1808         file = bdrv_open_child(filename, options, "file", bs,
	1809                                &child_file, true, &local_err);
	1810         if (local_err) {
	1811             goto fail;
	1812         }
	1813     }
	......	
	1846     /* Open the image */
	1847     ret = bdrv_open_common(bs, file, options, &local_err);
	1848     if (ret < 0) {
	1849         goto fail;
	1850     }
	......
	1912     return bs;
	1913
	......
	1933 }

1720~1730行如果reference非空的话，表明相应的bs数据结构已经创建，需要通过查找来获取相关bs的引用，通过bdrv\_open函数调用进入bdrv\_open\_inherit时，该参数为空，因此不会进入if分支内部。1739行，reference为空的话那么通过调用bdrv\_new（）函数来创建一个bs结构，该函数只是分配变量的内存空间，成员参数全部初始化为0，并将该bs结果插入到全局静态队里all\_bdrv\_states中。1752~1758行，如果child\_role非空的话则表示bdrv\_open\_inherit创建的是child类型的bs，因此设置该bs的inherits\_from，同时设置修改flags的值，使得修改后flags & BDRV_O_PROTOCOL不等于0。1788~1796行根据drvname 查找获取到对应drv的引用，这里获取到drv只是用于做一些判断，并不会调用drv的回调函数。1806~1813行，判断flags & BDRV\_O\_PROTOCOL是否等于0 ，如果等于零的话说明是在创建父bs，然后需要通过调用bdrv\_open\_child创建父bs的结构成员file，bdrv\_open\_child创建一个BdrvChildRole结构，同时会通过再次调用bdrv\_open\_inherit来创建BdrvChildRole的成员bs。1846~1850行，调用bdrv\_open\_common函数，bdrv\_open\_common会进一步调用特定镜像对应的bdrv所注册的回调函数来执行跟特定进行镜像相关的初始化工作，比如针对file 类型，会打开响应的镜像文件并将fd保存到bs相关的机构成员中，1912~1933行，返回创建的bs。



玩的开心 !!!
