

### parser



```C
const ParserApi api_h264d_parser = {
    .name = "h264d_parse",
    .coding = MPP_VIDEO_CodingAVC,
    .ctx_size = sizeof(H264_DecCtx_t),
    .flag = 0,
    .init = h264d_init,
    .deinit = h264d_deinit,
    .prepare = h264d_prepare,
    .parse = h264d_parse,
    .reset = h264d_reset,
    .flush = h264d_flush,
    .control = h264d_control,
    .callback = h264d_callback,
};
```

#### prepare

user传进来的数据都是封装在一个个pakcet里，而一个packet里的数据有可能就是一个完整的帧，也有可能不是完整的帧。

所以我们parser内部里收到数据后要进行分帧，分成一个完整的帧后再进行解析。而是否要进行分帧是根据need_split来判断的（由user来进行设置的）。

```C
        if (p_Inp->init.need_split) {
            ret = parse_prepare(p_Inp, p_Dec->p_Cur);
        } else {
            ret = parse_prepare_fast(p_Inp, p_Dec->p_Cur);
        }
```
prepare里会把一帧里所有的NALU保存下来，p_Cur->strm->head_buf[]



#### parse

一帧完整的数据是由多个NALU单元构成的，而这里会去循环解析每个NALU数据。

解析sps，pps，sei，slice信息。最后会把所有信息都保存到dxva里去。

#### init_picture()

这个函数做的事情比较多，需要好好看看

```C
MPP_RET init_picture(H264_SLICE_t *currSlice)
```

在解析完sps、pps、slice后，会调用init_picture函数，那里面具体做什么呢？

1. 创建对象H264_StorePic_t *dec_pic，并将slice里的各个信息保存到dec_pic里。

   - （1）这里还有一个动作是根据sps->pic_order_cnt_type去计算POC：decode_poc(p_Vid, currSlice)
   - （2）dpb_mark_malloc 申请新的frame，然后设置想要的flag和信息，挂到frames_slots上，

   ```C
   FUN_CHECK(ret = alloc_decpic(currSlice));
   static MPP_RET alloc_decpic(H264_SLICE_t *currSlice)
{
   ...
       dec_pic = alloc_storable_picture(p_Vid, currSlice->structure);
   ...
       FUN_CHECK(ret = decode_poc(p_Vid, currSlice));  //!< calculate POC (1)
   ...
       dec_pic->combine_flag = get_filed_dpb_combine_flag(p_Dpb->last_picture, dec_pic);
       /* malloc dpb_memory */
       FUN_CHECK(ret = dpb_mark_malloc(p_Vid, dec_pic));//(2)
       FUN_CHECK(ret = check_dpb_discontinuous(p_Vid->last_pic, dec_pic, currSlice));
   ...
   }
   ```
   
   

#### dpb

```C
typedef struct h264_dpb_buf_t {
    RK_U32   size;
    RK_U32   used_size;
    RK_U32   ref_frames_in_buffer;
    RK_U32   ltref_frames_in_buffer;
    RK_U32   used_size_il;

    RK_S32   poc_interval;
    RK_S32   last_output_poc;
    RK_S32   last_output_view_id;
    RK_S32   max_long_term_pic_idx;
    RK_S32   init_done;
    RK_S32   num_ref_frames;
    RK_S32   layer_id;

    struct h264_frame_store_t  **fs;
    struct h264_frame_store_t  **fs_ref;
    struct h264_frame_store_t  **fs_ltref;
    struct h264_frame_store_t  **fs_ilref;   //!< inter-layer reference (for multi-layered codecs)
    struct h264_frame_store_t   *last_picture;

    struct h264d_video_ctx_t   *p_Vid;
} H264_DpbBuf_t;
```

h264d_dpb.c文件主要就是管理这个dbp结构体对象的,管理参考帧的一个对象。介绍一下这几个成员含义：

- size

  dbp里可以存储的参考帧的最大长度，sps里的语法max_num_ref_frames，有指定了最大的参考帧数目，另外h264协议里规定了最大为16.
  
- used_size

  dbp里已经存储的参考帧数量。

- ref_frames_in_buffer

  短期参考帧的数目，短期参考帧,遇到下个I(IDR)会清空缓存

- ltref_frames_in_buffer

  长期参考帧的数目

- poc_interval

- last_output_poc

  上一帧输出帧对应的poc

- num_ref_frames

  短期参考帧的数目

- **fs

  具体存储参考帧的对象，包括短期和长期的

- **fs_ref

  存储短期参考帧对象

- **fs_ltref

  存储长期参考帧对象
  
- *last_picture

  如果当前帧为场编码，则用于存其中一场，等待下一场编码完成后合并

#### h264_frame_store_t

```C
//!< Frame Stores for Decoded Picture Buffer
typedef struct h264_frame_store_t {
    RK_S32    is_used;                //!< 0=empty; 1=top; 2=bottom; 3=both fields (or frame)
    RK_S32    is_reference;           //!< 0=not used for ref; 1=top used; 2=bottom used; 3=both fields (or frame) used
    RK_S32    is_long_term;           //!< 0=not used for ref; 1=top used; 2=bottom used; 3=both fields (or frame) used
    RK_S32    is_orig_reference;      //!< original marking by nal_ref_idc: 0=not used for ref; 1=top used; 2=bottom used; 3=both fields (or frame) used
    RK_S32    is_non_existent;
    RK_S32    frame_num_wrap;
    RK_S32    long_term_frame_idx;
    RK_S32    is_output;
    RK_S32    poc;
    RK_S32    view_id;
    RK_S32    inter_view_flag[2];
    RK_S32    anchor_pic_flag[2];
    RK_S32    layer_id;
    RK_S32    slice_type;
    RK_U32    frame_num;
    RK_S32    structure;
    RK_U32    is_directout;
    struct h264_store_pic_t *frame;
    struct h264_store_pic_t *top_field;
    struct h264_store_pic_t *bottom_field;
} H264_FrameStore_t;
```

这个结构体就是dbp里具体存储参考帧对象的结构体。

#### h264_store_pic_t

```C
//!< definition a picture (field or frame)
typedef struct h264_store_pic_t {
    RK_S32    structure;
    RK_S32    poc;
    RK_S32    top_poc;
    RK_S32    bottom_poc;
    RK_S32    frame_poc;
    RK_S32    ThisPOC;
    RK_S32    pic_num;
    RK_S32    long_term_pic_num;
    RK_S32    long_term_frame_idx;

    RK_U32    frame_num;
    RK_U8     is_long_term;
    RK_S32    used_for_reference;
    RK_S32    is_output;
    RK_S32    non_existing;
    RK_S16    max_slice_id;
    RK_S32    mb_aff_frame_flag;
    RK_U32    PicWidthInMbs;
    RK_U8     colmv_no_used_flag;
    struct h264_store_pic_t *top_field;     //!< for mb aff, if frame for referencing the top field
    struct h264_store_pic_t *bottom_field;  //!< for mb aff, if frame for referencing the bottom field
    struct h264_store_pic_t *frame;         //!< for mb aff, if field for referencing the combined frame

    RK_S32       slice_type;
    RK_S32       idr_flag;
    RK_S32       no_output_of_prior_pics_flag;
    RK_S32       long_term_reference_flag;
    RK_S32       adaptive_ref_pic_buffering_flag;
    RK_S32       chroma_format_idc;
    RK_S32       frame_mbs_only_flag;
    RK_S32       frame_cropping_flag;

    RK_S32       frame_crop_left_offset;
    RK_S32       frame_crop_right_offset;
    RK_S32       frame_crop_top_offset;
    RK_S32       frame_crop_bottom_offset;
    RK_S32       width;
    RK_S32       height;
    RK_U32       width_after_crop;
    RK_U32       height_after_crop;

    struct h264_drpm_t   *dec_ref_pic_marking_buffer;                    //!< stores the memory management control operations
    RK_S32       proc_flag;
    RK_S32       view_id;
    RK_S32       inter_view_flag;
    RK_S32       anchor_pic_flag;
    RK_S32       iCodingType;

    RK_S32       layer_id;
    RK_U8        is_mmco_5;
    RK_S32       poc_mmco5;
    RK_S32       top_poc_mmco5;
    RK_S32       bot_poc_mmco5;
    RK_S32       combine_flag;  // top && bottom field combined flag
    H264_Mem_type       mem_malloc_type;
    struct h264_dpb_mark_t     *mem_mark;
} H264_StorePic_t;
```



#### store_picture_in_dpb()

这个函数主要是将一个h264_store_pic_t对象保存到dbp，在里面会做一些更新的动作。

1. set use flag, 判断一下mem_mark是不是空，然后设置对应index的frame_slots的flag为SLOT_CODEC_USE

   ```C
       if (p->mem_mark && (p->mem_mark->slot_idx >= 0)) {
           mpp_buf_slot_set_flag(p_Vid->p_Dec->frame_slots, p->mem_mark->slot_idx, SLOT_CODEC_USE);
       } 
   ```

2. 判断这一帧是不是idr帧，是则更新dbp，不是idr，则判断adaptive_ref_pic_buffering_flag是否为1，则走adaptive_memory_management（自适应标记模式）还是sliding_window_memory_management（滑窗模式）

   ```C
       if (p->idr_flag) {
           FUN_CHECK(ret = idr_memory_management(p_Dpb, p));
       } else {    //!< adaptive memory management
           if (p->used_for_reference && p->adaptive_ref_pic_buffering_flag) {
               FUN_CHECK(ret = adaptive_memory_management(p_Dpb, p));
           }
       }
   ```

3. 从dbp中移除已经输出的参考帧

   ```C
   while (!remove_unused_frame_from_dpb(p_Dpb));
   ```

4. 当dbp中的缓存满了之后，会根据情况移除一帧。

   (1)移除已经输出的非参考帧

   (2)找到poc最小的帧

   (3)如果当前帧是非参考帧，且dpb中没有比当前帧更小的poc，则直接输出当前帧，不插入dpb。

   (4)如果当前是参考帧，且dpb没有比当前帧更小的poc，则输出poc最小的帧，并且从bpb移除这一帧。

   (5)从dpb中移除poc最小的帧

   ```C
       //!< when full output one frame
       while (p_Dpb->used_size >= p_Dpb->size) {
           RK_S32 min_poc = 0, min_pos = 0;
           RK_S32 find_flag = 0;
   
           remove_unused_frame_from_dpb(p_Dpb); //(1)
           find_flag = get_smallest_poc(p_Dpb, &min_poc, &min_pos);//(2)
           //(3)
           if (!p->used_for_reference) {
               if ((!find_flag) || (p->poc < min_poc)) {
                   FUN_CHECK(ret = direct_output(p_Vid, p_Dpb, p));  //!< output frame
                   goto __RETURN;
               }
           }
           //!< used for reference, but not find, then flush a frame in the first
           //(4)
           if ((!find_flag) || (p->poc < min_poc)) {
               //min_pos = 0;
               unmark_for_reference(p_Vid->p_Dec, p_Dpb->fs[min_pos]);
               if (!p_Dpb->fs[min_pos]->is_output) {
                   FUN_CHECK(ret = write_stored_frame(p_Vid, p_Dpb, p_Dpb->fs[min_pos]));
               }
               FUN_CHECK(ret = remove_frame_from_dpb(p_Dpb, min_pos));
               p->is_long_term = 0;
           }
           //(5)
           FUN_CHECK(ret = output_one_frame_from_dpb(p_Dpb));
       }
   ```

5. 将当前的参考帧保存到dbp中

   ```C
   FUN_CHECK(ret = insert_picture_in_dpb(p_Vid, p_Dpb->fs[p_Dpb->used_size], p, 0));
   ```

6. 更新短期和长期参考列表

   ```C
       scan_dpb_output(p_Dpb, p);
       update_ref_list(p_Dpb);
       update_ltref_list(p_Dpb);
   ```

   

### HAL硬件

MPP中和编解码硬件相关的配置是封装在对应的hal里。对于264解码，我们主要看如下：

<img src="https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201424.png" alt="image-20200622163340663" style="zoom:80%;" />

可以看到h264的解码有三个不同的硬件模块：rkv, vdpu1, vdpu2。rkv是rk自己研发的硬件编解码模块。

hal_h264d_api.c里定义了如下几个函数，上图几个文件的入口。

![image-20200622164233561](C:\Users\rockchip\AppData\Roaming\Typora\typora-user-images\image-20200622164233561.png)



- init主要是做一些初始化动作，在这里会决定是走哪个硬件模块，是rkv？vdpu1？还是vdpu2。

```C
MPP_RET hal_h264d_init(void *hal, MppHalCfg *cfg)
```

- 将parser解析出来的sps，pps等信息，配置到对应的寄存器里。

```C
MPP_RET hal_h264d_gen_regs(void *hal, HalTaskInfo *task)
{
    H264dHalCtx_t *p_hal = (H264dHalCtx_t *)hal;

    explain_input_buffer(hal, &task->dec);
    return p_hal->hal_api.reg_gen(hal, task);
}
```

- 将保存好各个信息的寄存器信息，通过ioctrl配置到driver。

```C
MPP_RET hal_h264d_start(void *hal, HalTaskInfo *task)
{
    H264dHalCtx_t *p_hal = (H264dHalCtx_t *)hal;
    return p_hal->hal_api.start(hal, task);
}
```

