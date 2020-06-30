



## MPP 学习

### mpp 框架

![image-20200605163700589](C:%5CUsers%5Crockchip%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20200605163700589.png)

![image-20200605162859128](C:%5CUsers%5Crockchip%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20200605162859128.png)

### Mpi

mpi主要就是暴露一套接口给外部去使用mpp内部的编解码。

初始化主要的func:

创建MpiImpl的一个结构体，传入的*ctx会返回一个MpiImpl对象，**mpi其中api就是mpi里一系列接口函数

```c++
MPP_RET mpp_create(MppCtx *ctx, MppApi **mpi)
```

初始化，这里面会去初始化mpp里的一系列管理buffer或者task的对象，以及初始化对应的enc或者dec的

```c
MPP_RET mpp_init(MppCtx ctx, MppCtxType type, MppCodingType coding)
```

mpi暴露给外部的一系列接口都会直接bypass call到对应Mpp里的接口。所以我们直接看mpp里的接口。

<img src="https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201213.png" alt="image-20200605094705421" style="zoom: 50%;" />

### mpp

编解码的过程对于外部来说主要是：

1. 喂buffer给编解码内部
2. 取出编解码好的buffer

#### 接口

那mpp里有提供了两套接口用于喂数据和取数据。

- 这一套就是我们常见的dequeue和enqueue，内部会维护一个queue来存放buffer。这个decode和encode都可以使用这套接口

```C++
MPP_RET poll(MppPortType type, MppPollType timeout);
MPP_RET dequeue(MppPortType type, MppTask *task);
MPP_RET enqueue(MppPortType type, MppTask task);
```

- 这两个是用于decode的时候使用的接口，put和get主要是操作mpp的mPackets和mFrames

```C++
MPP_RET put_packet(MppPacket packet);
MPP_RET get_frame(MppFrame *frame);
```

- 这两个是用于encode时候使用的接口，虽然它是put和get，但是它内部实现也是走上面的enqueue和dequeue，只是封装了下。

```C++
MPP_RET put_frame(MppFrame frame);
MPP_RET get_packet(MppPacket *packet);
```

#### 数据成员

mpp在init的时候会创建如下一些成员，来做具体的一些buffer和task的管理。

```C++
    mpp_list        *mPackets;
    mpp_list        *mFrames;
    mpp_list        *mTimeStamps;
    
	MppPort         mInputPort;
    MppPort         mOutputPort;

    MppTaskQueue    mInputTaskQueue;
    MppTaskQueue    mOutputTaskQueue;
```

#### control

mpp还提供了一个control接口，主要是用于配置一些编解码参数使用的，可以在init前以及runtime去设置。

```C++
MPP_RET control(MpiCmd cmd, MppParam param);
```

MpiCmd里定义了一系列cmd用于control去设置，有些则需要在init前设置才有效，因为是init时候需要用到。

```C++
typedef enum {
    MPP_OSAL_CMD_BASE                   = CMD_MODULE_OSAL,
    MPP_OSAL_CMD_END,

    MPP_CMD_BASE                        = CMD_MODULE_MPP,
    MPP_ENABLE_DEINTERLACE,
    MPP_SET_INPUT_BLOCK,                /* deprecated */
    MPP_SET_INTPUT_BLOCK_TIMEOUT,       /* deprecated */
    MPP_SET_OUTPUT_BLOCK,               /* deprecated */
    MPP_SET_OUTPUT_BLOCK_TIMEOUT,       /* deprecated */
    /*
     * timeout setup, refer to  MPP_TIMEOUT_XXX
     * zero     - non block
     * negative - block with no timeout
     * positive - timeout in milisecond
     */
    MPP_SET_INPUT_TIMEOUT,              /* parameter type RK_S64 */
    MPP_SET_OUTPUT_TIMEOUT,             /* parameter type RK_S64 */
    MPP_CMD_END,

    MPP_CODEC_CMD_BASE                  = CMD_MODULE_CODEC,
    MPP_CODEC_GET_FRAME_INFO,
    MPP_CODEC_CMD_END,

    MPP_DEC_CMD_BASE                    = CMD_MODULE_CODEC | CMD_CTX_ID_DEC,
    MPP_DEC_SET_FRAME_INFO,             /* vpu api legacy control for buffer slot dimension init */
    MPP_DEC_SET_EXT_BUF_GROUP,          /* IMPORTANT: set external buffer group to mpp decoder */
    MPP_DEC_SET_INFO_CHANGE_READY,
    MPP_DEC_SET_PRESENT_TIME_ORDER,     /* use input time order for output */
    MPP_DEC_SET_PARSER_SPLIT_MODE,      /* Need to setup before init */
    MPP_DEC_SET_PARSER_FAST_MODE,       /* Need to setup before init */
    MPP_DEC_GET_STREAM_COUNT,
    MPP_DEC_GET_VPUMEM_USED_COUNT,
    MPP_DEC_SET_VC1_EXTRA_DATA,
    MPP_DEC_SET_OUTPUT_FORMAT,
    MPP_DEC_SET_DISABLE_ERROR,          /* When set it will disable sw/hw error (H.264 / H.265) */
    MPP_DEC_SET_IMMEDIATE_OUT,
    MPP_DEC_SET_ENABLE_DEINTERLACE,     /* MPP enable deinterlace by default. Vpuapi can disable it */
    MPP_DEC_CMD_END,

    MPP_ENC_CMD_BASE                    = CMD_MODULE_CODEC | CMD_CTX_ID_ENC,
    /* basic encoder setup control */
    MPP_ENC_SET_CFG,                    /* set MppEncCfg structure */
    MPP_ENC_GET_CFG,                    /* get MppEncCfg structure */
    MPP_ENC_SET_PREP_CFG,               /* deprecated set MppEncPrepCfg structure, use MPP_ENC_SET_CFG instead */
    MPP_ENC_GET_PREP_CFG,               /* deprecated get MppEncPrepCfg structure, use MPP_ENC_GET_CFG instead */
    MPP_ENC_SET_RC_CFG,                 /* deprecated set MppEncRcCfg structure, use MPP_ENC_SET_CFG instead */
    MPP_ENC_GET_RC_CFG,                 /* deprecated get MppEncRcCfg structure, use MPP_ENC_GET_CFG instead */
    MPP_ENC_SET_CODEC_CFG,              /* deprecated set MppEncCodecCfg structure, use MPP_ENC_SET_CFG instead */
    MPP_ENC_GET_CODEC_CFG,              /* deprecated get MppEncCodecCfg structure, use MPP_ENC_GET_CFG instead */
    /* runtime encoder setup control */
    MPP_ENC_SET_IDR_FRAME,              /* next frame will be encoded as intra frame */
    MPP_ENC_SET_OSD_LEGACY_0,           /* deprecated */
    MPP_ENC_SET_OSD_LEGACY_1,           /* deprecated */
    MPP_ENC_SET_OSD_LEGACY_2,           /* deprecated */
    MPP_ENC_GET_HDR_SYNC,               /* get vps / sps / pps which has better sync behavior parameter is MppPacket */
    MPP_ENC_GET_EXTRA_INFO,             /* deprecated */
    MPP_ENC_SET_SEI_CFG,                /* SEI: Supplement Enhancemant Information, parameter is MppSeiMode */
    MPP_ENC_GET_SEI_DATA,               /* SEI: Supplement Enhancemant Information, parameter is MppPacket */
    MPP_ENC_PRE_ALLOC_BUFF,             /* allocate buffers before encoding */
    MPP_ENC_SET_QP_RANGE,               /* used for adjusting qp range, the parameter can be 1 or 2 */
    MPP_ENC_SET_ROI_CFG,                /* set MppEncROICfg structure */
    MPP_ENC_SET_CTU_QP,                 /* for H265 Encoder,set CTU's size and QP */

    /* User define rate control stategy API control */
    MPP_ENC_CFG_RC_API                  = CMD_MODULE_CODEC | CMD_CTX_ID_ENC | CMD_ENC_CFG_RC_API,
    /*
     * Get RcApiQueryAll structure
     * Get all available rate control stategy string and count
     */
    MPP_ENC_GET_RC_API_ALL              = MPP_ENC_CFG_RC_API + 1,
    /*
     * Get RcApiQueryType structure
     * Get available rate control stategy string with certain type
     */
    MPP_ENC_GET_RC_API_BY_TYPE          = MPP_ENC_CFG_RC_API + 2,
    /*
     * Set RcImplApi structure
     * Add new or update rate control stategy function pointers
     */
    MPP_ENC_SET_RC_API_CFG              = MPP_ENC_CFG_RC_API + 3,
    /*
     * Get RcApiBrief structure
     * Get current used rate control stategy brief information (type and name)
     */
    MPP_ENC_GET_RC_API_CURRENT          = MPP_ENC_CFG_RC_API + 4,
    /*
     * Set RcApiBrief structure
     * Set current used rate control stategy brief information (type and name)
     */
    MPP_ENC_SET_RC_API_CURRENT          = MPP_ENC_CFG_RC_API + 5,

    MPP_ENC_CFG_FRM                     = CMD_MODULE_CODEC | CMD_CTX_ID_ENC | CMD_ENC_CFG_FRM,
    MPP_ENC_SET_FRM,                    /* set MppFrame structure */
    MPP_ENC_GET_FRM,                    /* get MppFrame structure */


    MPP_ENC_CFG_PREP                    = CMD_MODULE_CODEC | CMD_CTX_ID_ENC | CMD_ENC_CFG_PREP,
    MPP_ENC_SET_PREP,                   /* set MppEncPrepCfg structure */
    MPP_ENC_GET_PREP,                   /* get MppEncPrepCfg structure */

    MPP_ENC_CFG_H264                    = CMD_MODULE_CODEC | CMD_CTX_ID_ENC | CMD_ENC_CFG_H264,

    MPP_ENC_CFG_H265                    = CMD_MODULE_CODEC | CMD_CTX_ID_ENC | CMD_ENC_CFG_H265,

    MPP_ENC_CFG_VP8                     = CMD_MODULE_CODEC | CMD_CTX_ID_ENC | CMD_ENC_CFG_VP8,

    MPP_ENC_CFG_MJPEG                   = CMD_MODULE_CODEC | CMD_CTX_ID_ENC | CMD_ENC_CFG_MJPEG,

    MPP_ENC_CFG_MISC                    = CMD_MODULE_CODEC | CMD_CTX_ID_ENC | CMD_ENC_CFG_MISC,
    MPP_ENC_SET_HEADER_MODE,            /* set MppEncHeaderMode */
    MPP_ENC_GET_HEADER_MODE,            /* get MppEncHeaderMode */

    MPP_ENC_CFG_SPLIT                   = CMD_MODULE_CODEC | CMD_CTX_ID_ENC | CMD_ENC_CFG_SPLIT,
    MPP_ENC_SET_SPLIT,                  /* set MppEncSliceSplit structure */
    MPP_ENC_GET_SPLIT,                  /* get MppEncSliceSplit structure */

    MPP_ENC_CFG_GOPREF                  = CMD_MODULE_CODEC | CMD_CTX_ID_ENC | CMD_ENC_CFG_GOPREF,
    MPP_ENC_SET_GOPREF,                 /* set MppEncGopRef structure */

    MPP_ENC_CFG_OSD                     = CMD_MODULE_CODEC | CMD_CTX_ID_ENC | CMD_ENC_CFG_OSD,
    MPP_ENC_SET_OSD_PLT_CFG,            /* set OSD palette, parameter should be pointer to MppEncOSDPltCfg */
    MPP_ENC_GET_OSD_PLT_CFG,            /* get OSD palette, parameter should be pointer to MppEncOSDPltCfg */
    MPP_ENC_SET_OSD_DATA_CFG,           /* set OSD data with at most 8 regions, parameter should be pointer to MppEncOSDData */
    MPP_ENC_GET_OSD_DATA_CFG,           /* get OSD data with at most 8 regions, parameter should be pointer to MppEncOSDData */

    MPP_ENC_CMD_END,

    MPP_ISP_CMD_BASE                    = CMD_MODULE_CODEC | CMD_CTX_ID_ISP,
    MPP_ISP_CMD_END,

    MPP_HAL_CMD_BASE                    = CMD_MODULE_HAL,
    MPP_HAL_CMD_END,

    MPI_CMD_BUTT,
} MpiCmd;
```



### MppEnc

MppEnc的init的时候，主要init如下两个：

```C++
    ret = mpp_enc_hal_init(&enc_hal, &enc_hal_cfg);
    ret = enc_impl_init(&impl, &ctrl_cfg);
```

![image-20200605164251339](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201220.png)



### MppDec

MppDec的init主要通过如下两个api来创建parser和halImpl

```C++
        ret = mpp_parser_init(&parser, &parser_cfg);
        ret = mpp_hal_init(&hal, &hal_cfg);
```

![image-20200605165348955](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201233.png)



### TaskQueue/Task

Mpp提供的dequeue/enqueue接口里就是操作MppTaskQueue，会有两个：一个Output，一个Input。

然后MppTaskQueue是如何管理Task的呢？

MppTaskQueue相关的关系如下：

![image-20200605105253279](C:\Users\rockchip\AppData\Roaming\Typora\typora-user-images\image-20200605105253279.png)

#### MppTaskQueue

1. task_count: 记录该queue里一共有多少个task，setup的时候确定好。
2. ready: 表示该queue已经setup过了
3. info[ ]: 记录各个status的info，一开始setup queue的时候，所有的task状态都为MPP_INPUT_PORT,所以也只有info[MPP_INPUT_PORT]->count大于0
4. tasks：在queue里轮转的tasks，默认情况下enc只有一个task，dec的时候jpeg也只有一个，而其他codec会有4个。



#### MppTaskStatusInfo

- 也就是MppTaskQueue里的info[ ],通过不同status的info来连接处于不同status下的MppTask 
- 一开始setup queue的时候，所有的task状态都为MPP_INPUT_PORT,所以也只有info[MPP_INPUT_PORT]->count大于0

1. count: 是个计数器，记录处于该status下的task有多少个。在enqueue和dequeue的时候会更新count
2. status:记录这个info是什么status的info
3. list: 连接该status下不同的MppTask

```C++
typedef struct MppTaskStatusInfo_t {
    struct list_head    list;
    RK_S32              count;
    MppTaskStatus       status;
    Condition           *cond;
} MppTaskStatusInfo;
```



#### MppTask

list: 通过这个来挂在MppTaskStatusInfo里的list下

status:记录该task的状态，会有两个状态：

PORT: task在queue里可以被dequeue拿去用。

HOLD：task已经被dequeue出去正在处理中

```C++
typedef enum MppTaskStatus_e {
    MPP_INPUT_PORT,             /* in external queue and ready for external dequeue */
    MPP_INPUT_HOLD,             /* dequeued and hold by external user, user will config */
    MPP_OUTPUT_PORT,            /* in mpp internal work queue and ready for mpp dequeue */
    MPP_OUTPUT_HOLD,            /* dequeued and hold by mpp internal worker, mpp is processing */
    MPP_TASK_STATUS_BUTT,
} MppTaskStatus;
```

目前enc的时候，setup taskqueue时候默认只创建一个task，并且status为MPP_INPUT_PORT，等待被dequeue.



#### MppPort

1. MppPort也有两个input和output，在初始化的定义好status_curr,next_ondequeue,next_on_enqueue的status，通过这个来定位MppTaksQueue里不同status的info。
2. 在通过info来找到对应status的MppTask。

```C++
//init MppPort 
if (MPP_PORT_INPUT == type) {
        impl->status_curr     = MPP_INPUT_PORT;
        impl->next_on_dequeue = MPP_INPUT_HOLD;
        impl->next_on_enqueue = MPP_OUTPUT_PORT;
    } else {
        impl->status_curr     = MPP_OUTPUT_PORT;
        impl->next_on_dequeue = MPP_OUTPUT_HOLD;
        impl->next_on_enqueue = MPP_INPUT_PORT;
    }
```



#### MppMeta

- 多个不同的MppMetaNode都串在list_node的后面
- node_count: 计数器，记录MppMetaNode的个数

```C++
typedef struct MppMetaImpl_t {
    char                tag[MPP_TAG_SIZE];
    const char          *caller;
    RK_S32              meta_id;
    RK_S32              ref_count;

    struct list_head    list_meta;
    struct list_head    list_node;
    RK_S32              node_count;
} MppMetaImpl;
```



### MppPacket/MppFrame

具体封装buffer的还有两个不同的对象

- MppPaket： 这个主要是存储一维数据，也就是编码后的数据，例如H264,H265..
- MppFrame：这个主要是存储二维数据，也就是解码后的数据，例如YUV..

而上面所说的MppTask又是如何和这些具体存储数据的对象联系在一块呢？

就是通过MppMetaNode来连接的，那我们就来看看MppNode相关的对象是怎样的。

![image-20200605114351039](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201244.png)

#### MppMetaNode

- MppMetaNode会有一个单例的MppMetaService来管理。
- 而MppMetaNode和MppTask的联系就是通过MppTask里的MppMeta来连接的，通过其中的list来串起来，一个MppMeta会连接多个MppMetaNode。

#### MppMetaVal

这是挂在MppMetaNode下，从其成员来看，就知道这里会存储MppFrame和MppPacket对象。



### buffer的申请

- buffer的申请最终还是要通过对应的allocator去做。
- 每一块buffer的申请都是通过MppBufferGroup里的allocator去申请的，然后由MppBufferGroup来管理。只是申请来的buffer会挂到frame或者packet里。
- 可以看到MppBufferGroup也是由一个单例的MppBufferService来管理的。
- MppAllocator里有个os_api，是根据不同系统

<img src="C:\Users\rockchip\AppData\Roaming\Typora\typora-user-images\image-20200605141124416.png" alt="image-20200605141124416" style="zoom:80%;" />

#### MppBufferGroup

1、group_id: 区分group用的，累加的方式来作为id。

2、limit_count: MppBuffer的最大限制数

3、buffer_count: 已有的MppBuffer的数量

3、count_used/count_unsed: 记录已用和未用buffer的数量

4、list_used/list_unused: 链表，链接已用和未用的MppBuffer(buffer->list_status)

5、usage: 记录所有MppBuffer的size之和



#### MppAllocator

os_api: 根据不同的bufferType,走不同的buffer allocator：

```
allocator_ion
allocator_drm
allocator_std
allocator_ext_dma
```

### MPP dec start

![dec start](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201253.png)

### MPP dec runtime flow



![dec runtime](E:\work\astah\dec runtime.png)

### Mpp enc start

![encod start](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201301.png)

### Mpp enc runtime flow

<img src="https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201316.png" alt="enc runtime" style="zoom:150%;" />