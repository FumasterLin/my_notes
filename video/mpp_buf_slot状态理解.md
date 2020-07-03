## mpp_buf_slot简介

mpp_buf_slot主要是在解码的时候使用，管理结构体对象：MppBufSlotsImpl_t

- MppBufSlotsImpl_t会有多个MppBufSlotEntry_t对象，每个MppBufSlotEntry_t会存储MppFrame和对应的MppBuffer。也就是buffer会挂在MppBufSlotEntry里，然后MppBufSlots会管理多个MppBufSlotEntry

- 正常我们解码会申请两个MppBufSlotsImpl_t对象: frame_slots和packet_slots

- 我们解码的buffer会挂到这个MppBufSlots里去管理，都是通过index去访问对应的slot

![image-20200702175207759](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200702175215.png)

MppBufSlotEntry_t

```C
//关于slot使用的状态操作
typedef enum SlotUsageType_e {
    SLOT_CODEC_READY,       // bit flag             for buffer is prepared by codec
    SLOT_CODEC_USE,         // bit flag             for buffer is used as reference by codec
    SLOT_HAL_INPUT,         // counter              for buffer is used as hardware input
    SLOT_HAL_OUTPUT,        // counter + bit flag   for buffer is used as hardware output
    SLOT_QUEUE_USE,         // bit flag             for buffer is hold in different queues
    SLOT_USAGE_BUTT,
} SlotUsageType;
//关于slot buffer设置的操作
typedef enum SlotPropType_e {
    SLOT_EOS,
    SLOT_FRAME,
    SLOT_BUFFER,
    SLOT_FRAME_PTR,
    SLOT_PROP_BUTT,
} SlotPropType;
```



上面的几个flag都有对应set和clr的操作，对应会改变如下几个状态，这几个是标记MppBufSlotEntry的状态：

```C
typedef union SlotStatus_u {
    RK_U32 val;
    struct {
        // status flags
        RK_U32  on_used     : 1;
        RK_U32  not_ready   : 1;        // buffer slot is filled or not
        RK_U32  codec_use   : 1;        // buffer slot is used by codec ( dpb reference )
        RK_U32  hal_output  : 1;        // buffer slot is set to hw output will ready when hw done
        RK_U32  hal_use     : 8;        // buffer slot is used by hardware
        RK_U32  queue_use   : 5;        // buffer slot is used in different queue

        // value flags
        RK_U32  eos         : 1;        // buffer slot is last buffer slot from codec
        RK_U32  has_buffer  : 1;
        RK_U32  has_frame   : 1;
    };
} SlotStatus;
```

## 关于slot使用的状态操作

如下的操作使用的接口：mpp_buf_slot_set_flag / mpp_buf_slot_clr_flag

1. **SLOT_CODEC_READY**
   - 对应改变not_ready的状态，set的时候为0，clr为1。
   - 在设置SLOT_HAL_OUTPUT的时候也会相应更新not_ready的状态。
2. **SLOT_CODEC_USE**
   - 对应改变codec_use的状态，set置为1，clr置为0.
   - 一般创建的新的MppFrame挂到slot里去后就会被置上，在移除参考的时候clear这个状态
   - 这个set和clr的操作都是在parser里做的。
3. **SLOT_HAL_INPUT**
   - 对应改变hal_use的状态，hal_use是个计数器，每次设置SLOT_HAL_INPUT的时候+1，清除SLOT_HAL_INPUT的时候减1。
   - 表示这个buffer正在作为input，被硬件使用中
4. **SLOT_HAL_OUTPUT**
   - 对应改变hal_output的状态，not_ready也会随着更新， set置为1，clr置为0。
   - 表示这个buffer正在作为output，被硬件使用中
5. **SLOT_QUEUE_USE**
   - 对应改变queue_use的状态，queue_use是个计数器，每次设置SLOT_QUEUE_USE的时候加1，清除SLOT_QUEUE_USE的时候减1.
   - 表示这个buffer被放到其他的queue里去了，目前parser里都是enqueue到display queue里。

## 关于slot设置的操作

如下的操作使用的接口：mpp_buf_slot_set_prop / mpp_buf_slot_get_prop

1. **SLOT_FRAME**

   - mpp_buf_slot_set_prop 

     > 设置MppFrame到slot里，此时slot会申请一个新的MppFrame，然后通过memcpy将传进来的MppFrame val内存复制进来。
     >
     > status.has_frame = 1

   - mpp_buf_slot_get_prop

     > 将对应slot里的MppFrame内容memcpy给传进来的MppFrame val

2. **SLOT_BUFFER**

   - mpp_buf_slot_set_prop 

     > 外部设置MppBuffer地址到slot->buffer，buffer是通过MppBufferGroup去申请的
     >
     > 当slot里有MppFrame的时候，同时也会把buffer地址设置到MppFrame里，所以PacketSlots没有这个步骤
     >
     > status.has_buffer = 1

   - mpp_buf_slot_get_prop

     > 获取MppBuffer地址，slot->buffer

3. **SLOT_FRAME_PTR**

   - mpp_buf_slot_get_prop

     > 获取MppFrame地址，slot->frame

## 以VP8解码为例讲解SlotEntry状态更新过程

### packet_slots:

#### 设置过程

##### set SLOT_SET_ON_USE / SLOT_SET_NOT_READY

mpp_dec.cpp里的parser_thread

当parser去调用mpp_buf_slot_get_unused获取未使用的MppBufSlotEntry的index来准备使用的时候，会标记如下两个状态：

- set SLOT_SET_ON_USE : on_used = 1

- set SLOT_SET_NOT_READY : not_ready = 1

##### set SLOT_CODEC_READY / SLOT_HAL_INPUT 

mpp_dec.cpp里try_proc_dec_task里将码流复制到packet_slots里的时候，会将对应的slot设置：

- set SLOT_CODEC_READY : not_ready = 0

- set SLOT_HAL_INPUT : hal_use ++

```C
    if (!task->status.dec_pkt_copy_rdy) {
        void *dst = mpp_buffer_get_ptr(task->hal_pkt_buf_in);
        void *src = mpp_packet_get_data(task_dec->input_packet);
        size_t length = mpp_packet_get_length(task_dec->input_packet);

        memcpy(dst, src, length);
        mpp_buf_slot_set_flag(packet_slots, task_dec->input, SLOT_CODEC_READY);//
        mpp_buf_slot_set_flag(packet_slots, task_dec->input, SLOT_HAL_INPUT);//
        task->status.dec_pkt_copy_rdy = 1;
    }
```

#### 清除过程

##### clear SLOT_HAL_INPUT 

hal_thread:

等硬件做完之后，清除SLOT_HAL_INPUT

```C
mpp_buf_slot_clr_flag(packet_slots, task_dec->input,SLOT_HAL_INPUT);
```

- clear SLOT_HAL_INPUT : hal_use--



### frame_slots：

#### 设置过程

vp8d_parser.c里的vp8d_alloc_frame里：

##### set SLOT_SET_ON_USE / SLOT_SET_NOT_READY

1. 获取未使用slot的index,设置SLOT_SET_ON_USE 和SLOT_SET_NOT_READY 

   ```C
   mpp_buf_slot_get_unused(p->frame_slots,&p->frame_out->slot_index);
   ```

   - set SLOT_SET_ON_USE : on_used = 1

   - set SLOT_SET_NOT_READY : not_ready = 1

##### set SLOT_CODEC_USE  / SLOT_HAL_OUTPUT

1. 设置MppFrame到slot后，设置SLOT_CODEC_USE 和SLOT_HAL_OUTPUT 

   ```C
   mpp_buf_slot_set_flag(p->frame_slots, p->frame_out->slot_index,SLOT_CODEC_USE);
   mpp_buf_slot_set_flag(p->frame_slots, p->frame_out->slot_index,SLOT_HAL_OUTPUT);
   ```

   - set SLOT_CODEC_USE : codec_use = 1

   - set SLOT_HAL_OUTPUT : hal_output = 1， not_ready = 1

##### set SLOT_CODEC_USE / enqueue QUEUE_DISPLAY

1. 如果当前帧是要显示的，设置SLOT_QUEUE_USE 和enqueue到QUEUE_DISPLAY 

   ```C
   if (p->showFrame) {
       mpp_buf_slot_set_flag(p->frame_slots, p->frame_out->slot_index,SLOT_QUEUE_USE);
       mpp_buf_slot_enqueue(p->frame_slots, p->frame_out->slot_index,QUEUE_DISPLAY);
   }
   ```
   - set SLOT_QUEUE_USE : queue_use++

   - enqueue QUEUE_DISPLAY :  queue_use++, 同时将对应的slot enqueue到display queue里

##### set SLOT_HAL_INPUT

vp8d_parser.c里的vp8d_convert_to_syntx

​	如果对应的参考帧不为空，则设置SLOT_HAL_INPUT，表示作为input给硬件使用

```C
    if (p->frame_ref != NULL) {
        pic_param->lst_fb_idx.Index7Bits = p->frame_ref->slot_index;
        mpp_buf_slot_set_flag(p->frame_slots, p->frame_ref->slot_index,
                              SLOT_HAL_INPUT);
        in_task->refer[0] = p->frame_ref->slot_index;
    } else {
        pic_param->lst_fb_idx.Index7Bits = 0x7f;
    }

    if (p->frame_golden != NULL) {
        pic_param->gld_fb_idx.Index7Bits = p->frame_golden->slot_index;
        mpp_buf_slot_set_flag(p->frame_slots, p->frame_golden->slot_index,
                              SLOT_HAL_INPUT);
        in_task->refer[1] = p->frame_golden->slot_index;
    } else {
        pic_param->gld_fb_idx.Index7Bits = 0x7f;
    }

    if (p->frame_alternate != NULL) {
        pic_param->alt_fb_idx.Index7Bits = p->frame_alternate->slot_index;
        mpp_buf_slot_set_flag(p->frame_slots, p->frame_alternate->slot_index,
                              SLOT_HAL_INPUT);
        in_task->refer[2] = p->frame_alternate->slot_index;
    } else {
        pic_param->alt_fb_idx.Index7Bits = 0x7f;
    }
```

- set SLOT_HAL_INPUT : hal_use++

#### 清除过程

##### Clear SLOT_CODEC_USE

vp8d_parser.c里的vp8d_ref_update里会去更新一些参考帧的信息，不参考的则会清除SLOT_CODEC_USE

```C
static void vp8d_unref_frame(VP8DParserContext_t *p, VP8Frame *frame)
{
...
    frame->ref_count--;
    if (!frame->ref_count && frame->slot_index < 0x7f) {
        mpp_buf_slot_clr_flag(p->frame_slots, frame->slot_index, SLOT_CODEC_USE);
        frame->slot_index = 0xff;
        mpp_free(frame->f);
        mpp_free(frame);
        frame = NULL;
    }
...
}

```

- clear SLOT_CODEC_USE : codec_use = 0

mpp_dec里的hal_thread:

等硬件做完之后：

##### Clear SLOT_HAL_INPUT

1. 把记录参考帧的对应的slot清除SLOT_HAL_INPUT

```
for (RK_U32 i = 0; i < MPP_ARRAY_ELEMS(task_dec->refer); i++) {
    RK_S32 index = task_dec->refer[i];
    if (index >= 0)
    mpp_buf_slot_clr_flag(frame_slots, index, SLOT_HAL_INPUT);
}
```

- clear SLOT_HAL_INPUT : hal_use--

##### Clear SLOT_HAL_OUTPUT

2. 硬件做完了，也就是已经输出到output frame了，就可以清除SLOT_HAL_OUTPUT

   ```C
   if (task_dec->output >= 0)
       mpp_buf_slot_clr_flag(frame_slots, task_dec->output, SLOT_HAL_OUTPUT);
   ```

   - SLOT_HAL_OUTPUT : hal_output = 0， not_ready = 0

##### dequeue QUEUE_DISPLAY / Clear SLOT_QUEUE_USE

3. 从display queue里取出来index出来，然后清除SLOT_QUEUE_USE

```C
while (MPP_OK == mpp_buf_slot_dequeue(frame_slots, &index, QUEUE_DISPLAY)) {
    /* deal with current frame */
    if (eos && mpp_slots_is_empty(frame_slots, QUEUE_DISPLAY))
        tmp.eos = 1;

    mpp_dec_put_frame(mpp, index, tmp);
    mpp_buf_slot_clr_flag(frame_slots, index, SLOT_QUEUE_USE);
}
```

- dequeue QUEUE_DISPLAY : queue_use --
- clear SLOT_QUEUE_USE : queue_use --

### 回收

如上的设置和清除过程，正常的一个slot状态更新就走了一遍了，每次设置清除状态的时候，都会去检查一下各个状态：

如果符合如下if里的条件的时候，就会：

1. SLOT_CLR_FRAME和SLOT_CLR_BUFFER的动作，也就是清除对应的has_frame和has_buffer的标志。
2. 清除SLOT_CLR_ON_USE，on_used  = 0

这样这个slot就又回到unuse里去，等待下一次被使用了。

```C
static void check_entry_unused(MppBufSlotsImpl *impl, MppBufSlotEntry *entry)
{
    SlotStatus status = entry->status;

    if (status.on_used &&
        !status.not_ready &&
        !status.codec_use &&
        !status.hal_output &&
        !status.hal_use &&
        !status.queue_use) {
        if (entry->frame) {
            slot_ops_with_log(impl, entry, SLOT_CLR_FRAME, entry->frame);
            mpp_frame_deinit(&entry->frame);
        }
        if (entry->buffer) {
            mpp_buffer_put(entry->buffer);
            slot_ops_with_log(impl, entry, SLOT_CLR_BUFFER, entry->buffer);
            entry->buffer = NULL;
        }

        slot_ops_with_log(impl, entry, SLOT_CLR_ON_USE, NULL);
        impl->used_count--;
    }
}
```

