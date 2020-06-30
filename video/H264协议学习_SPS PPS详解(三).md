# SPS PPS详解

（转载自 https://zhuanlan.zhihu.com/p/27896239）

- 1 SPS和PPS从何处而来？
- 2 SPS和PPS中的每个参数起什么作用？
- 3 如何解析SDP中包含的H.264的SPS和PPS串？

##  SPS语法元素及其含义

在H.264标准协议中规定了多种不同的NAL
Unit类型，其中类型7表示该NAL Unit内保存的数据为Sequence Paramater
Set。在H.264的各种语法元素中，SPS中的信息至关重要。如果其中的数据丢失或出现错误，那么解码过程很可能会失败。SPS及后续将要讲述的图像参数集PPS在某些平台的视频处理框架（比如iOS的VideoToolBox等）还通常作为解码器实例的初始化信息使用。

SPS即Sequence Paramater Set，又称作序列参数集。SPS中保存了一组编码视频序列(Coded video sequence)的全局参数。所谓的编码视频序列即原始视频的一帧一帧的像素数据经过编码之后的结构组成的序列。而每一帧的编码后数据所依赖的参数保存于图像参数集中。一般情况SPS和PPS的NAL
Unit通常位于整个码流的起始位置。但在某些特殊情况下，在码流中间也可能出现这两种结构，主要原因可能为：

- 解码器需要在码流中间开始解码；
- 编码器在编码的过程中改变了码流的参数（如图像分辨率等）；

在做视频播放器时，为了让后续的解码过程可以使用SPS中包含的参数，必须对其中的数据进行解析。其中H.264标准协议中规定的SPS格式位于文档的7.3.2.1.1部分，如下图所示：

![img](https://pic2.zhimg.com/80/v2-5b11cd2e81c4b134730c9856468a4f69_720w.jpg)

其中的每一个语法元素及其含义如下：

(1) **profile_idc**：

标识当前H.264码流的profile。我们知道，H.264中定义了三种常用的档次profile：

基准档次：baseline profile;

主要档次：main profile;

扩展档次：extended profile;

在H.264的SPS中，第一个字节表示profile_idc，根据profile_idc的值可以确定码流符合哪一种档次。判断规律为：

profile_idc = 66 → baseline profile;

profile_idc = 77 → main profile;

profile_idc = 88 → extended profile;

在新版的标准中，还包括了High、High 10、High 4:2:2、High 4:4:4、High 10 Intra、High 4:2:2 Intra、High 4:4:4 Intra、CAVLC 4:4:4 Intra等，每一种都由不同的profile_idc表示。

另外，constraint_set0_flag ~ constraint_set5_flag是在编码的档次方面对码流增加的其他一些额外限制性条件。

在我们实验码流中，profile_idc = 0x42 = 66，因此码流的档次为baseline profile。

(2) **level_idc**

标识当前码流的Level。编码的Level定义了某种条件下的最大视频分辨率、最大视频帧率等参数，码流所遵从的level由level_idc指定。

当前码流中，level_idc = 0x1e = 30，因此码流的级别为3。

(3) **seq_parameter_set_id**

表示当前的序列参数集的id。通过该id值，图像参数集pps可以引用其代表的sps中的参数。

(4) **chroma_format_idc**

表 6-1－由chroma_format_idc决定的SubWidthC和SubHeightC的值

| chroma_format_idc | 色彩格式 | SubWidthC | SubHeightC |
| ----------------- | -------- | --------- | ---------- |
| 0                 | 单色     | -         | -          |
| 1                 | 4:2:0    | 2         | 2          |
| 2                 | 4:2:2    | 2         | 1          |
| 3                 | 4:4:4    | 1         | 1          |

(5) **log2_max_frame_num_minus4**

用于计算MaxFrameNum的值。计算公式为MaxFrameNum = 2^(log2_max_frame_num_minus4 +4)。MaxFrameNum是frame_num的上限值，frame_num是图像序号的一种表示方法，在帧间编码中常用作一种参考帧标记的手段。

(6) **pic_order_cnt_type**

表示解码picture order count(POC)的方法。POC是另一种计量图像序号的方式，与frame_num有着不同的计算方法。该语法元素的取值为0、1或2。

(7) **log2_max_pic_order_cnt_lsb_minus4**

用于计算MaxPicOrderCntLsb的值，该值表示POC的上限。计算方法为MaxPicOrderCntLsb = 2^(log2_max_pic_order_cnt_lsb_minus4 + 4)。

(8) **max_num_ref_frames**

用于表示参考帧的最大数目。

(9) **gaps_in_frame_num_value_allowed_flag**

标识位，说明frame_num中是否允许不连续的值。

(10) **pic_width_in_mbs_minus1**

用于计算图像的宽度。单位为宏块个数，因此图像的实际宽度为:

frame_width = 16 × (pic\_width\_in\_mbs_minus1 + 1);

(11) **pic_height_in_map_units_minus1**

使用PicHeightInMapUnits来度量视频中一帧图像的高度。PicHeightInMapUnits并非图像明确的以像素或宏块为单位的高度，而需要考虑该宏块是帧编码或场编码。PicHeightInMapUnits的计算方式为：

PicHeightInMapUnits = pic\_height\_in\_map\_units\_minus1 + 1;

(12) **frame_mbs_only_flag**

标识位，说明宏块的编码方式。当该标识位为0时，宏块可能为帧编码或场编码；该标识位为1时，所有宏块都采用帧编码。根据该标识位取值不同，PicHeightInMapUnits的含义也不同，为0时表示一场数据按宏块计算的高度，为1时表示一帧数据按宏块计算的高度。

按照宏块计算的图像实际高度FrameHeightInMbs的计算方法为：

FrameHeightInMbs = ( 2 − frame_mbs_only_flag ) * PicHeightInMapUnits

(13) **mb_adaptive_frame_field_flag**

标识位，说明是否采用了宏块级的帧场自适应编码。当该标识位为0时，不存在帧编码和场编码之间的切换；当标识位为1时，宏块可能在帧编码和场编码模式之间进行选择。

(14) **direct_8x8_inference_flag**

标识位，用于B_Skip、B_Direct模式运动矢量的推导计算。

(15) **frame_cropping_flag**

标识位，说明是否需要对输出的图像帧进行裁剪。

(16) **vui_parameters_present_flag**

标识位，说明SPS中是否存在VUI信息。

## PPS语法元素及其含义

除了序列参数集SPS之外，H.264中另一重要的参数集合为图像参数集Picture Paramater Set(PPS)。通常情况下，PPS类似于SPS，在H.264的裸码流中单独保存在一个NAL Unit中，只是PPS NAL Unit的nal_unit_type值为8；而在封装格式中，PPS通常与SPS一起，保存在视频文件的文件头中。

在H.264的协议文档中，PPS的结构定义在7.3.2.2节中，具体的结构如下表所示：

![img](https://pic1.zhimg.com/80/v2-0c288e37cefb04f54dc202159729f95c_720w.jpg)

其中的每一个语法元素及其含义如下：

(1) **pic_parameter_set_id**

表示当前PPS的id。某个PPS在码流中会被相应的slice引用，slice引用PPS的方式就是在Slice header中保存PPS的id值。该值的取值范围为[0,255]。

(2) **seq_parameter_set_id**

表示当前PPS所引用的激活的SPS的id。通过这种方式，PPS中也可以取到对应SPS中的参数。该值的取值范围为[0,31]。

(3) **entropy_coding_mode_flag**

熵编码模式标识，该标识位表示码流中熵编码/解码选择的算法。对于部分语法元素，在不同的编码配置下，选择的熵编码方式不同。例如在一个宏块语法元素中，宏块类型mb_type的语法元素描述符为“ue(v) | ae(v)”，在baseline profile等设置下采用指数哥伦布编码，在main profile等设置下采用CABAC编码。

标识位entropy_coding_mode_flag的作用就是控制这种算法选择。当该值为0时，选择左边的算法，通常为指数哥伦布编码或者CAVLC；当该值为1时，选择右边的算法，通常为CABAC。

(4) **bottom_field_pic_order_in_frame_present_flag**

标识位，用于表示另外条带头中的两个语法元素delta_pic_order_cnt_bottom和delta_pic_order_cn是否存在的标识。这两个语法元素表示了某一帧的底场的POC的计算方法。

(5) **num_slice_groups_minus1**

表示某一帧中slice group的个数。当该值为0时，一帧中所有的slice都属于一个slice group。slice group是一帧中宏块的组合方式，定义在协议文档的3.141部分。

(6) **num_ref_idx_l0_default_active_minus1**、**num_ref_idx_l1_default_active_minus1**

表示当Slice Header中的num_ref_idx_active_override_flag标识位为0时，P/SP/B
slice的语法元素num_ref_idx_l0_active_minus1和num_ref_idx_l1_active_minus1的默认值。

(7) **weighted_pred_flag**

标识位，表示在P/SP slice中是否开启加权预测。

(8) **weighted_bipred_idc**

表示在B Slice中加权预测的方法，取值范围为[0,2]。0表示默认加权预测，1表示显式加权预测，2表示隐式加权预测。

(9) **pic_init_qp_minus26**和**pic_init_qs_minus26**

表示初始的量化参数。实际的量化参数由该参数、slice header中的slice_qp_delta/slice_qs_delta计算得到。

(10) **chroma_qp_index_offset**

用于计算色度分量的量化参数，取值范围为[-12,12]。

(11) **deblocking_filter_control_present_flag**

标识位，用于表示Slice header中是否存在用于去块滤波器控制的信息。当该标志位为1时，slice header中包含去块滤波相应的信息；当该标识位为0时，slice header中没有相应的信息。

(12) **constrained_intra_pred_flag**

若该标识为1，表示I宏块在进行帧内预测时只能使用来自I和SI类型宏块的信息；若该标识位0，表示I宏块可以使用来自Inter类型宏块的信息。

(13) **redundant_pic_cnt_present_flag**

标识位，用于表示Slice header中是否存在redundant_pic_cnt语法元素。当该标志位为1时，slice header中包含redundant_pic_cnt；当该标识位为0时，slice header中没有相应的信息。