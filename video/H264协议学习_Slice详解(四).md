### 片头语义(slice_header)

| slice_header( ) {                                            | C    | 描述符 |
| ------------------------------------------------------------ | ---- | ------ |
| first_mb_in_slice                                            | 2    | ue(v)  |
| slice_type                                                   | 2    | ue(v)  |
| pic_parameter_set_id                                         | 2    | ue(v)  |
| frame_num                                                    | 2    | u(v)   |
| if( !frame_mbs_only_flag ) {                                 |      |        |
| field_pic_flag                                               | 2    | u(1)   |
| if( field_pic_flag )                                         |      |        |
| bottom_field_flag                                            | 2    | u(1)   |
| }                                                            |      |        |
| if( nal_unit_type = = 5 )                                    |      |        |
| idr_pic_id                                                   | 2    | ue(v)  |
| if( pic_order_cnt_type = = 0 ) {                             |      |        |
| pic_order_cnt_lsb                                            | 2    | u(v)   |
| if( pic_order_present_flag && !field_pic_flag )              |      |        |
| delta_pic_order_cnt_bottom                                   | 2    | se(v)  |
| }                                                            |      |        |
| if( pic_order_cnt_type = = 1 && !delta_pic_order_always_zero_flag ) { |      |        |
| delta_pic_order_cnt[ 0 ]                                     | 2    | se(v)  |
| if( pic_order_present_flag && !field_pic_flag )              |      |        |
| delta_pic_order_cnt[ 1 ]                                     | 2    | se(v)  |
| }                                                            |      |        |
| if( redundant_pic_cnt_present_flag )                         |      |        |
| redundant_pic_cnt                                            | 2    | ue(v)  |
| if( slice_type = = B )                                       |      |        |
| direct_spatial_mv_pred_flag                                  | 2    | u(1)   |
| if( slice_type = = P \| \| slice_type = = SP \| \| slice_type = = B ) { |      |        |
| num_ref_idx_active_override_flag                             | 2    | u(1)   |
| if( num_ref_idx_active_override_flag ) {                     |      |        |
| num_ref_idx_l0_active_minus1                                 | 2    | ue(v)  |
| if( slice_type = = B )                                       |      |        |
| num_ref_idx_l1_active_minus1                                 | 2    | ue(v)  |
| }                                                            |      |        |
| }                                                            |      |        |
| ref_pic_list_reordering( )                                   | 2    |        |
| if( ( weighted_pred_flag && ( slice_type = = P \| \| slice_type = = SP ) ) \| \| ( weighted_bipred_idc = = 1 && slice_type = = B ) ) |      |        |
| pred_weight_table( )                                         | 2    |        |
| if( nal_ref_idc != 0 )                                       |      |        |
| dec_ref_pic_marking( )                                       | 2    |        |
| if( entropy_coding_mode_flag && slice_type != I && slice_type != SI ) |      |        |
| cabac_init_idc                                               | 2    | ue(v)  |
| slice_qp_delta                                               | 2    | se(v)  |
| if( slice_type = = SP \| \| slice_type = = SI ) {            |      |        |
| if( slice_type = = SP )                                      |      |        |
| sp_for_switch_flag                                           | 2    | u(1)   |

| slice_qs_delta                                               | 2    | se(v) |
| ------------------------------------------------------------ | ---- | ----- |
| }                                                            |      |       |
| if( deblocking_filter_control_present_flag ) {               |      |       |
| disable_deblocking_filter_idc                                | 2    | ue(v) |
| if( disable_deblocking_filter_idc != 1 ) {                   |      |       |
| slice_alpha_c0_offset_div2                                   | 2    | se(v) |
| slice_beta_offset_div2                                       | 2    | se(v) |
| }                                                            |      |       |
| }                                                            |      |       |
| if( num_slice_groups_minus1 > 0 && slice_group_map_type >= 3 && slice_group_map_type <= 5) |      |       |
| slice_group_change_cycle                                     | 2    | u(v)  |
| }                                                            |      |       |

**first_mb_in_slice** 片中的第一个宏块的地址, 片通过这个句法元素来标定它自己的地址。要注意的是在帧场自适应模式下，宏块都是成对出现，这时本句法元素表示的是第几个宏块对，对应的第一个宏块的真实地址应该是：2 * first_mb_in_slice

**slice_type** 指明片的类型，具体语义见表7.21。
																													表 7.21 slice_type 语义

| slice_type | Name of slice_type |
| ---------- | ------------------ |
| 0          | P (P slice)        |
| 1          | B (B slice)        |
| 2          | I (I slice)        |
| 3          | SP (SP slice)      |
| 4          | SI (SI slice)      |
| 5          | P (P slice)        |
| 6          | B (B slice)        |
| 7          | I (I slice)        |
| 8          | SP (SP slice)      |
| 9          | SI (SI slice)      |

IDR 图像时, slice_type 等于 2, 4, 7, 9。
**pic_parameter_set_id**  图像参数集的索引号. 范围 0 到 255。
**frame_num** 每个参考帧都有一个依次连续的 frame_num 作为它们的标识,这指明了各图像的解码顺序。但事实上我们在表 中可以看到， frame_num 的出现没有 if 语句限定条件，这表明非参考帧的片头也会出现 frame_num。只是当该个图像是参考帧时，它所携带的这个句法元素在解码时才有意义。
如表 7.21 所示例子：
																					表 7.21 某个序列的 frame_num 值
														（该序列中 B 帧一律不作为参考帧，而 P 帧一律作为参考帧。 ）

| 图像序号 | 图像类型 | 是否用作参考 | frame_num |
| -------- | -------- | ------------ | --------- |
| 1        | I        | 是           | 0         |
| 2        | P        | 是           | 1         |
| 3        | B        | 否           | 2         |
| 4        | P        | 是           | 2         |
| 5        | B        | 否           | 3         |
| 6        | P        | 是           | 3         |
| 7        | B        | 否           | 4         |
| 8        | P        | 是           | 4         |
| …        | …        | …            | …         |

H.264 对 frame_num 的 值 作 了 如 下 规 定 ： 当 参 数 集 中 的 句 法 元 素gaps_in_frame_num_value_allowed_flag 不为 1 时，每个图像的 frame_num 值是它前一个参考帧的frame_num 值增加 1。这句话包含有两层意思：

1） 当 gaps_in_frame_num_value_allowed_flag 不为 1，即 frame_num 连续的情况下，每个图像的frame_num 由前一个参考帧图像对应的值加 1，着重点是“前一个参考帧”。在表 7.21 中第 3 个图像是 B 帧，按照定义，它的 frame_nun 值应是前一个参考帧，即第 2 个图像对应的值加 1，即为 2；第4 个图像是 P 帧，由于该序列 B 帧都不作为参考帧，所以对于该图像来说，定义中所谓的“前一个参考帧”，仍旧是指的第 2 个图像，所以对于第 4 个图像来说，它的 frame_num 的取值和第 3 个图像一样，也为 2。相同的情况也发生在第 6 和第 8 帧上。前面我们曾经提到，对于非参考帧来说，它的 frame_num 值在解码过程中是没有意义的，因为frame_num 值是参考帧特有的，它的主要作用是在该图像被其他图像引用作运动补偿的参考时提供一个标识。但 H.264 并没有在非参考帧图像中取消这一句法元素，原因是在 POC 的第二种和第三种解码方法中可以通过非参考帧的 frame_num 值计算出他们的 POC 值， 在第八章中会详细讲述这个问题。

2） 当 gaps_in_frame_num_value_allowed_flag 等于 1，前文已经提到，这时若网络阻塞，编码器可以将编码后的若干图像丢弃，而不用另行通知解码器。在这种情况下，解码器必须有机制将缺失的frame_num 及所对应的图像填补，否则后续图像若将运动矢量指向缺失的图像将会产生解码错误。

**field_pic_flag** 这是在片层标识图像编码模式的唯一一个句法元素。 所谓的编码模式是指的帧编码、场编码、帧场自适应编码。当这个句法元素取值为 1 时 属于场编码； 0 时为非场编码。序列参数集中的句法元素 frame_mbs_only_flag 和 mb_adaptive_frame_field_flag 再加上本句法元素共同决定图像的编码模式，如图 7.7 所示。
![image-20200623141526831](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629200047.png)
在序列参数集中我们已经能够计算出图像的高和宽的大小，但曾强调那里的高是指的该序列中图像的帧的高度，而一个实际的图像可能是帧也可能是场，对于图像的实际高度，应进一步作如下处理：
PicHeightInMbs = FrameHeightInMbs / ( 1 + field_pic_flag )
从而我们可以得到在解码器端所用到的其他与图像大小有关的变量：
PicHeightInSamplesL = PicHeightInMbs * 16
PicHeightInSamplesC = PicHeightInMbs * 8
PicSizeInMbs = PicWidthInMbs * PicHeightInMbs
前文已提到， frame_num 是参考帧的标识，但是在解码器中，并不是直接引用的 frame_num 值，而是由 frame_num 进一步计算出来的变量 PicNum,在第八章会详细讲述由 frame_num 映射到PicNum 的算法。这里介绍在该算法中用到的两个变量。
MaxPicNum：
表征 PicNum 的最大值， PicNum 和 frame_num 一样，也是嵌在循环中，当达到这个最大值时，
PicNum 将从 0 开始重新计数。
\- 如果field_pic_flag= 0, MaxPicNum = MaxFrameNum.
\- 否则， MaxPicNum =2*MaxFrameNum.
CurrPicNum：
当前图像的 PicNum 值，在计算 PicNum 的过程中，当前图像的 PicNum 值是由 frame_num 直接算出（在第八章中会看到， 在解某个图像时， 要将已经解码的各参考帧的 PicNum 重新计算一遍，新的值参考当前图像的 PicNum 得来）。
\- 如果field_pic_flag= 0， CurrPicNum = frame_num.
\- 否则, CurrPicNum= 2 * frame_num + 1.
Frame_num是对帧编号的，也就是说如果在场模式下，同属一个场对的顶场和底场两个图像的frame_num的值是相同的。在帧或帧场自适应模式下，就直接将图像的frame_num赋给PicNum，而在场模式下， 将2 * frame_num 和 2 * frame_num + 1两个值分别赋给两个场。 2 * frame_num + 1这个值永远被赋给当前场，解码到当前场对的下一个场时，刚才被赋为2 * frame_num + 1的场的PicNum值被重新计算为2 * frame_num ，而将2 * frame_num + 1赋给新的当前场。

**bottom_field_flag** 等于 1 时表示当前图像是属于底场；等于 0 时表示当前图像是属于顶场。idr_pic_id IDR 图像的标识。不同的 IDR 图像有不同的 idr_pic_id 值。值得注意的是， IDR 图像有不等价于 I 图像，只有在作为 IDR 图像的 I 帧才有这个句法元素，在场模式下， IDR 帧的两个场有相同的 idr_pic_id 值。 idr_pic_id 的取值范围是 [0， 65535]，和 frame_num 类似，当它的值超出这个范围时，它会以循环的方式重新开始计数。

**pic_order_cnt_lsb** 在 POC 的第一种算法中本句法元素来计算 POC 值，在 POC 的第一种算法中是显式地传递 POC 的值，而其他两种算法是通过 frame_num 来映射 POC 的值。注意这个句法元素的读取函数是 u(v),这个 v 的来自是图像参数集：
v=log2_max_pic_order_cnt_lsb_minus4 + 4。

**delta_pic_order_cnt_bottom** 如果是在场模式下，场对中的两个场都各自被构造为一个图像，它们有各自的 POC 算法来分别计算两个场的 POC 值，也就是一个场对拥有一对 POC 值；而在是帧模式或是帧场自适应模式下， 一个图像只能根据片头的句法元素计算出一个 POC 值。 根据 H.264 的规定，在序列中有可能出现场的情况，即 frame_mbs_only_flag 不为 1 时，每个帧或帧场自适应的图像在解码完后必须分解为两个场，以供后续图像中的场作为参考图像。

所以当 frame_mb_only_flag 不为 1时，帧或帧场自适应中包含的两个场也必须有各自的 POC 值。在第八章中我们会看到，通过本句法元素，可以在已经解开的帧或帧场自适应图像的 POC 基础上新映射一个 POC 值，并把它赋给底场。当然，象句法表指出的那样，这个句法元素只用在 POC 的第一个算法中。

**delta_pic_order_cnt[ 0 ]， delta_pic_order_cnt[ 1 ]：**
前文已经提到， POC 的第二和第三种算法是从 frame_num 映射得来，这两个句法元素用于映射算法。 delta_pic_order_cnt[ 0 ]用于帧编码方式下的底场和场编码方式的场， delta_pic_order_cnt[ 1 ]用于帧编码方式下的顶场。在第八章会详细讲述 POC 的三种算法。

**redundant_pic_cnt** 冗余片的 id 号。

**direct_spatial_mv_pred_flag** 指出在B图像的直接预测的模式下，用时间预测还是用空间预测。 1：空间预测； 0：时间预测。

**num_ref_idx_active_override_flag** 在 图 像 参 数 集 中 我 们 看 到 已 经 出 现 句 法 元 素num_ref_idx_l0_active_minus1 和 num_ref_idx_l1_active_minus1 指定当前参考帧队列中实际可用的参考帧的数目。在片头可以重载这对句法元素，以给某特定图像更大的灵活度。这个句法元素就是指明片头是否会重载，如果该句法元素等于 1，下面会出现新的 num_ref_idx_l0_active_minus1 和num_ref_idx_l1_active_minus1 值。

**num_ref_idx_l0_active_minus1 、 num_ref_idx_l1_active_minus1**
如 上 个 句 法 元 素 中 所 介 绍 ， 这 是 重 载 的 num_ref_idx_l0_active_minus1 及num_ref_idx_l1_active_minus1

**cabac_init_idc** 给出 cabac 初始化时表格的选择，范围 0 到 2。

**slice_qp_delta** 指出在用于当前片的所有宏块的量化参数的初始值。 QPY																																		    								SliceQPY = 26 + pic_init_qp_minus26 + slice_qp_delta
QPY 的范围是 0 to 51。
我们前文已经提到， H.264 中量化参数是分图像参数集、片头、宏块头三层给出的，前两层各自给出一个偏移值，这个句法元素就是片层的偏移。

**sp_for_switch_flag** 指出 SP 帧中的 p 宏块的解码方式是否是 switching 模式， 在第八章有详细的说明。

**slice_qs_delta** 与 slice_qp_delta 的与语义相似，用在 SI 和 SP 中的

​												QSY = 26 + pic_init_qs_minus26 + slice_qs_delta
QSY 值的范围是 0 到 51。

**disable_deblocking_filter_idc** H.264 指定了一套算法可以在解码器端独立地计算图像中各边界的滤波强度进行滤波。除了解码器独立计算之外，编码器也可以传递句法元素来干涉滤波强度，当这个句法元素指定了在块的边界是否要用滤波，同时指明那个块的边界不用块滤波，在第八章会详细讲述。 .

**slice_alpha_c0_offset_div2** 给出用于增强 α 和 tC0 的偏移值
												FilterOffsetA = slice_alpha_c0_offset_div2 << 1
slice_alpha_c0_offset_div2 值的范围是 -6 到+6。

**slice_beta_offset_div2** 给出用于增强 β 和 tC0 的偏移值
											  FilterOffsetB = slice_beta_offset_div2 << 1
slice_beta_offset_div2 值的范围是 -6 到+6。

**slice_group_change_cycle** 当片组的类型是 3, 4, 5，由句法元素可获得片组中 映射单元的数目：

More ActionsMapUnitsInSliceGroup0 = Min(  PicSizeInMapUnits )slice_group_change_cycle * SliceGroupChangeRate,

slice_group_change_cycle 由 Ceil( Log2( PicSizeInMapUnits ÷ SliceGroupChangeRate + 1 ) )位比特表示
slice_group_change_cycle 值的范围是 0 到 Ceil( PicSizeInMapUnits÷SliceGroupChangeRate )。

### 参考图像序列重排序的语义(ref_pic_list_recordering)

![image-20200623142629757](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629200056.png)

每一个使用帧间预测的图像都会引用前面已解码的图像作为参考帧。如前文所述，编码器给每个参考帧都会分配一个唯一性的标识，即句法元素 frame_num。但是，当编码器要指定当前图像的参考图像时，并不是直接指定该图像的 frame_num 值，而是使用通过下面步骤最终得出的 ref_id 号：

![image-20200623142730403](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629200108.png)
其中， 从 frame_num 到变换到变量 PicNum 主要是考虑到场模式的需要，当序列中允许出现场时，每个非场的图像（帧或 帧场自适应）解码后必须分解为一个场对，从而需要为它们分解后的两个场各自指定一个标识； 进一步从 PicNum 到 ref_id 是为了能节省码流， 因为 PicNum 的值通常都比较大，而在帧间预测时，需要为每个运动矢量都指明相对应的参考帧的标识，如果这个标识选用 PicNum开销就会比较大，所以 H.264 又将 PicNum 映射为一个更小的变量 ref_id。在编码器和解码器都同步地维护一个参考帧队列,每解码一个片就将该队列刷新一次,把各图像按照特定的规则进行排序,排序后各图像在队列中的序号就是该图像的 ref_id 值,在下文中我们可以看到在宏块层表示参考图像的标识就是 ref_id, 在第八章我们会详细讲述队列的初始化、排序等维护算法。

本节和下节介绍的是在维护队列时两个重要操作：重排序(reordering)和标记(marking)所用到的句法元素。说明：句法元素的后缀名带有 L0 指的是第一个参数列表；句法元素的后缀名带有 L1 指的是第二个参数列表（用在 B 帧预测中）。

**ref_pic_list_reordering_flag_l0** 指明是否进行重排序操作,这个句法元素等于 1 时表明紧跟着会有一系列句法元素用于参考帧队列的重排序。

**re_pic_list_reordering_flag_l1** 与上个句法元素的语义相同，只是本句法元素用于参考帧队列 L1。

**reordering_of_pic_nums_idc** 指明执行哪种重排序操作,具体语义见表 7.22。
																										表 7.22 重排序操作

| reordering_of_pic_nums_idc | 操作                                                         |
| -------------------------- | ------------------------------------------------------------ |
| 0                          | 短期参考帧重排序， abs_diff_pic_num_minus1会出现在码流中，从当 前图像的PicNum减去 (abs_diff_pic_num_minus1 + 1) 后指明需要重 排序的图像。 |
| 1                          | 短期参考帧重排序， abs_diff_pic_num_minus1会出现在码流中，从当 前图像的PicNum加上 (abs_diff_pic_num_minus1 + 1) 后指明需要重 排序的图像。 |
| 2                          | 长期参考帧重排序， long_term_pic_num会出现在码流中，指明需要重 排序的图像。 |
| 3                          | 结束循环，退出重排序操作。                                   |

**abs_diff_pic_num_minus1** 在对短期参考帧重排序时指明重排序图像与当前的差。见表 7.22。

**long_term_pic_num** 在对长期参考帧重排序时指明重排序图像。见表 7.22。
从 上 文 可 以 看 到 ， 每 组 reordering_of_pic_nums_idc 、 abs_diff_pic_num_minus1 或reordering_of_pic_nums_idc、 long_term_pic_num 只能对一个图像操作，而通常情况下都需要对一组图像重排序，所以在码流中一般会有个循环，反复出现这些重排序的句法元素，如图，循环直到reordering_of_pic_nums_id 等于 3 结束，我们在表 7 .22 中也可以看到这种情况。
![image-20200623143554183](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629200118.png)

### 加权预测的语义(pred_weight_table)

![image-20200623143651368](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629200130.png)

**luma_log2_weight_denom** 给 出 参 考 帧 列 表 中 参 考 图 像 所 有 亮 度 的 加 权 系 数 ， 是 个 初 始 值luma_log2_weight_denom 值的范围是 0 to 7。

**chroma_log2_weight_denom** 给 出 参 考 帧 列 表 中 参 考 图 像 所 有 色 度 的 加 权 系 数 ， 是 个 初 始 值chroma_log2_weight_denom 值的范围是 0 to 7。

**luma_weight_l0_flag** 等于 1 时，指的是在参考序列 0 中的亮度的加权系数存在；等于 0 时，在参考序列 0 中的亮度的加权系数不存在。

**luma_weight_l0[ i ]** 用参考序列 0 预测亮度值时，所用的加权系数。如果 luma_weight_l0_flag is = 0,luma_weight_l0[ i ] = 2luma_log2_weight_denom 。

**luma_offset_l0[ i ]** 用参考序列 0 预测亮度值时， 所用的加权系数的额外的偏移。 luma_offset_l0[ i ] 值的范围–128 to 127。如果 luma_weight_l0_flag is = 0, luma_offset_l0[ i ] = 0 。

**chroma_weight_l0_flag， chroma_weight_l0[ i ]\[ j ] ， chroma_offset_l0[ i ]\[ j ]** 与上述三个类似，不同的是这三个句法元素是用在色度预测。

**luma_weight_l1_flag, luma_weight_l1, luma_offset_l1, chroma_weight_l1_flag, chroma_weight_l1,chroma_offset_l1**
与 luma_weight_l0_flag, luma_weight_l0, luma_offset_l0, chroma_weight_l0_flag, chroma_weight_l0,chroma_offset_l0 的语义相似，不同点是这几个句法元素是用在参考序列 1 中。

### 参考图像序列标记 (marking)操作的语义(dec_ref_pic_marking)

![image-20200623144114231](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629200144.png)

前文介绍的重排序（ reordering）操作是对参考帧队列重新排序， 而标记（ marking）操作负责将参考图像移入或移出参考帧队列。

**no_output_of_prior_pics_flag** 仅在当前图像是 IDR 图像时出现这个句法元素， 指明是否要将前面已解码的图像全部输出。

**long_term_reference_flag** 与上个图像一样，仅在当前图像是 IDR 图像时出现这一句法元素。这个句法元素指明是否使用长期参考这个机制。 如果取值为 1，表明使用长期参考，并且每个 IDR 图像被解码后自动成为长期参考帧，否则（取值为 0）， IDR 图像被解码后自动成为短期参考帧。

**adaptive_ref_pic_marking_mode_flag** 指明标记（ marking）操作的模式，具体语义见表 7.23。
																			表 7.23 标记（ marding）模式

| adaptive_ref_pic_marking_mode_flag | 标记（ marking）模式                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| 0                                  | 先入先出（ FIFO）：使用滑动窗的机制，先入先出，在这种模式 下没有办法对长期参考帧进行操作。 |
| 1                                  | 自适应标记（ marking）：后续码流中会有一系列句法元素显式指 明操作的步骤。自适应是指编码器可根据情况随机灵活地作出决 策。 |

**memory_management_control_operation** 在自适应标记（ marking）模式中，指明本次操作的具体内容，具体语义见表 7.24。
																				表 7.24 标记（ marking）操作

| memory_management_control_operation | 标记（ marking）操作                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| 0                                   | 结束循环，退出标记（ marding）操作。                         |
| 1                                   | 将一个短期参考图像标记为非参考图像，也 即将一个短期参考图像移出参考帧队列。 |
| 2                                   | 将一个长期参考图像标记为非参考图像，也 即将一个长期参考图像移出参考帧队列。 |
| 3                                   | 将一个短期参考图像转为长期参考图像。                         |
| 4                                   | 指明长期参考帧的最大数目。                                   |
| 5                                   | 清空参考帧队列，将所有参考图像移出参考 帧队列，并禁用长期参考机制 |
| 6                                   | 将当前图像存为一个长期参考帧。                               |

**difference_of_pic_nums_minus1** 当 memory_management_control_operation 等于 3 或 1 时， 由 这个句法元素可以计算得到需要操作的图像在短期参考队列中的序号。 参考帧队列中必须存在这个图像。

**long_term_pic_num** 当 memory_management_control_operation 等于 2 时， 从此句法元素得到所要操作的长期参考图像的序号。

**long_term_frame_idx** 当 memory_management_control_operation 等于 3 或 6 ，分配一个长期参考帧的序号给一个图像。

**max_long_term_frame_idx_plus1** 此 句 法 元 素 减 1 ， 指 明 长 期 参 考 队 列 的 最 大 数 目 。

**max_long_term_frame_idx_plus1** 值的范围 0 to num_ref_frames。