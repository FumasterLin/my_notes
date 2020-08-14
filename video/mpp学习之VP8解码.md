1. frame_slots在vp8的parser里初始化15个MppBufSlotEntry，也就是input buffer可以有15帧在轮转int

```C
MPP_RET vp8d_parser_init(void *ctx, ParserCfg *parser_cfg)
{
...
    mpp_buf_slot_setup(p->frame_slots, 15);
...
}
```

## 对照vp8标准Chapter 9: Frame Header  和mpp vp8 parser 实现

VP8的码流解析主要在decoder_frame_header里(如下函数)，其中还有调用vp8_header_parser，最主要还是vp8_header_parser里做解析的动作。

```C
static MPP_RET decoder_frame_header(VP8DParserContext_t *p, RK_U8 *pbase, RK_U32 size)
{
..
    p->keyFrame = !(pbase[0] & 1);
    p->vpVersion = (pbase[0] >> 1) & 7;
    p->showFrame = 1;
...
    if (p->decMode == VP8HWD_VP7) {
        p->offsetToDctParts = (pbase[0] >> 4) | (pbase[1] << 4) | (pbase[2] << 12);
        p->frameTagSize = p->vpVersion >= 1 ? 3 : 4;
    } else {
        p->offsetToDctParts = (pbase[0] >> 5) | (pbase[1] << 3) | (pbase[2] << 11);
        p->showFrame = (pbase[0] >> 4) & 1;
        p->frameTagSize = 3;
    }
    pbase += p->frameTagSize;
    size -= p->frameTagSize;
    if (p->keyFrame)
        vp8hwdResetProbs(p);
    //mpp_log_f("p->decMode = %d", p->decMode);
    if (p->decMode == VP8HWD_VP8) {
        ret = vp8_header_parser(p, pbase, size);
    }  else {
        ret = vp7_header_parser(p, pbase, size);
    }
...
}
```

### 9.1 Uncompressed Data Chunk

那我们现在根据VP8 标准来分析code。

![image-20200629115140546](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629194938.png)

前三个字节是frame tag：

**keyframe** 第1个bit 

**verision number** 接着后面3个bits

**show_frame** 第5个bit，指示这一帧是否需要显示

**offsetToDctParts** 剩下的19个bit，看文档 意思是第一个数据段的大小（bytes）

```C
    p->keyFrame = !(pbase[0] & 1);
    p->vpVersion = (pbase[0] >> 1) & 7;

    p->offsetToDctParts = (pbase[0] >> 5) | (pbase[1] << 3) | (pbase[2] << 11);
    p->showFrame = (pbase[0] >> 4) & 1;
    p->frameTagSize = 3;
```

剩下的，就要看vp8_header_parser里的解析了，也是对照文档来一个一个来看。

![image-20200629141249251](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201445.png)

如上图所说的，对于**keyframe**来说，还有7个字节的信息，分别对应三个字节的start code：0x9d 01 2a， 四个字节的宽高信息以及对应宽高缩放比例因子。

![image-20200629141539214](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201451.png)

具体如何读取出startCode以及宽高信息以及比例因子根据上图的示意，对应实现code如下：

```C
    if (p->keyFrame) {
        tmp = (pbase[0] << 16) | (pbase[1] << 8) | (pbase[2] << 0);
        if (tmp != VP8_KEY_FRAME_START_CODE)
            return MPP_ERR_PROTOL;
        tmp = (pbase[3] << 0) | (pbase[4] << 8);
        p->width = tmp & 0x3fff;
        p->scaledWidth = ScaleDimension(p->width, tmp >> 14);
        tmp = (pbase[5] << 0) | (pbase[6] << 8);
        p->height = tmp & 0x3fff;
        p->scaledHeight = ScaleDimension(p->height, tmp >> 14);
        pbase += 7;
        size -= 7;
    }
```

其中ScaleDimension()的实现如下，是根据标准中给的不同的比例因子对应不同的scale比例：

![image-20200629141945865](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201459.png)

```C
static RK_U32 ScaleDimension( RK_U32 orig, RK_U32 scale )
{
    FUN_T("FUN_IN");
    switch (scale) {
    case 0:
        return orig;
        break;
    case 1: /* 5/4 */
        return (5 * orig) / 4;
        break;
    case 2: /* 5/3 */
        return (5 * orig) / 3;
        break;
    case 3: /* 2 */
        return 2 * orig;
        break;
    }
    FUN_T("FUN_OUT");
    return orig;
}
```

### 9.2 Color Space and Pixel Type (Key Frames-only)  

![image-20200629142747562](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201508.png)

```C
    if (p->keyFrame) {
        p->colorSpace = (vpColorSpace_e)vp8hwdDecodeBool128(bit_ctx);
        p->clamping = vp8hwdDecodeBool128(bit_ctx);
    }
```

The color space type bit is encoded as the following:
• 0 - YUV color space similar to the YCrCb color space defined in ITU-R BT.601
• 1 - YUV color space whose digital conversion to RGB does not involve multiplication and division  

The pixel value clamping type bit is encoded as the following:
• 0 - Decoders are required to clamp the reconstructed pixel values to between 0 and 255
(inclusive).
• 1 - Reconstructed pixel values are guaranteed to be between 0 and 255 , no clamping is
necessary  

### 9.3 Segment-based Adjustments

![image-20200629144925672](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201523.png)

以上的文档描述对应如下的实现，序号对应标记

```C
    p->segmentationEnabled = vp8hwdDecodeBool128(bit_ctx); //(1) 只有这个flag为1的时候，才有后续的2-5的步骤
    p->segmentationMapUpdate = 0;
    if (p->segmentationEnabled) {
        p->segmentationMapUpdate = vp8hwdDecodeBool128(bit_ctx);//(2)
        if (vp8hwdDecodeBool128(bit_ctx)) {    /* Segmentation map update */ //(3)
            //if (3) is 1 then (4)
            p->segmentFeatureMode = vp8hwdDecodeBool128(bit_ctx);//(4.1)
            memset(&p->segmentQp[0], 0, MAX_NBR_OF_SEGMENTS * sizeof(RK_S32));
            memset(&p->segmentLoopfilter[0], 0,
                   MAX_NBR_OF_SEGMENTS * sizeof(RK_S32));
            //(4.2)
            for (i = 0; i < MAX_NBR_OF_SEGMENTS; i++) {
                if (vp8hwdDecodeBool128(bit_ctx)) {
                    p->segmentQp[i] = vp8hwdReadBits(bit_ctx, 7);
                    if (vp8hwdDecodeBool128(bit_ctx))
                        p->segmentQp[i] = -p->segmentQp[i];
                }
            }
            for (i = 0; i < MAX_NBR_OF_SEGMENTS; i++) {
                if (vp8hwdDecodeBool128(bit_ctx)) {
                    p->segmentLoopfilter[i] = vp8hwdReadBits(bit_ctx, 6);
                    if (vp8hwdDecodeBool128(bit_ctx))
                        p->segmentLoopfilter[i] = -p->segmentLoopfilter[i];
                }
            }
        }
        //(5)
        if (p->segmentationMapUpdate) {
            p->probSegment[0] = 255;
            p->probSegment[1] = 255;
            p->probSegment[2] = 255;
            for (i = 0; i < 3; i++) {
                if (vp8hwdDecodeBool128(bit_ctx)) {
                    p->probSegment[i] = vp8hwdReadBits(bit_ctx, 8);
                }
            }
        }
    }
```



### 9.4 Loop Filter Type and Levels  

![image-20200629150744843](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201532.png)

这三个字段读取如下，具体的含义解释需要看标准文档的第十五章

```C
    p->loopFilterType = vp8hwdDecodeBool128(bit_ctx);
    p->loopFilterLevel = vp8hwdReadBits(bit_ctx, 6);
    p->loopFilterSharpness = vp8hwdReadBits(bit_ctx, 3);
```

1. **modeRefLfEnabled** 接下来读取modeRefLfEnabled的标记，只有这个是1，才有后面的字段读取。

![image-20200629150925356](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201608.png)

2. **mbRefLfDelta[i] 和 mbModeLfDelta[i]** 的读取

![image-20200629151307343](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201616.png)

```C
    p->modeRefLfEnabled = vp8hwdDecodeBool128(bit_ctx);// 1.
    if (p->modeRefLfEnabled) {
        //2.
        if (vp8hwdDecodeBool128(bit_ctx)) {
            for (i = 0; i < MAX_NBR_OF_MB_REF_LF_DELTAS; i++) {
                if (vp8hwdDecodeBool128(bit_ctx)) {
                    p->mbRefLfDelta[i] = vp8hwdReadBits(bit_ctx, 6);
                    if (vp8hwdDecodeBool128(bit_ctx))
                        p->mbRefLfDelta[i] = -p->mbRefLfDelta[i];
                }
            }
            for (i = 0; i < MAX_NBR_OF_MB_MODE_LF_DELTAS; i++) {
                if (vp8hwdDecodeBool128(&p->bitstr)) {
                    p->mbModeLfDelta[i] = vp8hwdReadBits(bit_ctx, 6);
                    if (vp8hwdDecodeBool128(bit_ctx))
                        p->mbModeLfDelta[i] = -p->mbModeLfDelta[i];
                }
            }
        }
    }
```

### 9.5 Token Partition and Partition Data Offsets  

![image-20200629151746567](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201627.png)

```
p->nbrDctPartitions = vp8hwdReadBits(bit_ctx, 2);
```

### 9.6 Dequantization Indices

1. 

![image-20200629152020490](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201638.png)

以上的数据读取如下：

```C
    p->qpYAc = vp8hwdReadBits(bit_ctx, 7);
    p->qpYDc = DecodeQuantizerDelta(bit_ctx);
    p->qpY2Dc = DecodeQuantizerDelta(bit_ctx);
    p->qpY2Ac = DecodeQuantizerDelta(bit_ctx);
    p->qpChDc = DecodeQuantizerDelta(bit_ctx);
    p->qpChAc = DecodeQuantizerDelta(bit_ctx);
```

2. 

![image-20200629152052158](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201648.png)

以上的数据读取如下，就是上面的delta()部分的实现：

```C
static RK_S32 DecodeQuantizerDelta(vpBoolCoder_t *bit_ctx)
{
    RK_S32  result = 0;
    if (vp8hwdDecodeBool128(bit_ctx)) {
        result = vp8hwdReadBits(bit_ctx, 4);
        if (vp8hwdDecodeBool128(bit_ctx))
            result = -result;
    }
    return result;
}
```



### 9.7 Refresh Golden Frame and AltRef Frame

1. 

![image-20200629153051083](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201833.png)

- (1)对于keyframe来说，默认情况下，黄金帧和altref帧均由当前重构的帧刷新/替换。

- (2)对于非关键帧，VP8使用重构的当前帧使用两位指示两个帧缓冲区是否被刷新

  - (2.1)

    ![image-20200629153458241](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201844.png)

  - (2.2)

    ![image-20200629153601243](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201855.png)

  - (2.3)

    ![image-20200629153626380](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629201916.png)

  - (2.4)

    ![image-20200629153656725](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202056.png)

```C
    if (p->keyFrame) {//(1)
        p->refreshGolden          = 1;
        p->refreshAlternate       = 1;
        p->copyBufferToGolden     = 0;
        p->copyBufferToAlternate  = 0;

        /* Refresh entropy probs */
        p->refreshEntropyProbs = vp8hwdDecodeBool128(bit_ctx);

        p->refFrameSignBias[0] = 0;
        p->refFrameSignBias[1] = 0;
        p->refreshLast = 1;
    } else {//(2)
        //(2.1)
        /* Refresh golden */
        p->refreshGolden = vp8hwdDecodeBool128(bit_ctx);
        /* Refresh alternate */
        p->refreshAlternate = vp8hwdDecodeBool128(bit_ctx);
        //(2.2)
        if ( p->refreshGolden == 0 ) {
            /* Copy to golden */
            p->copyBufferToGolden = vp8hwdReadBits(bit_ctx, 2);
        } else
            p->copyBufferToGolden = 0;
		//(2.3)
        if ( p->refreshAlternate == 0 ) {
            /* Copy to alternate */
            p->copyBufferToAlternate = vp8hwdReadBits(bit_ctx, 2);
        } else
            p->copyBufferToAlternate = 0;
		//(2.3)
        /* Sign bias for golden frame */
        p->refFrameSignBias[0] = vp8hwdDecodeBool128(bit_ctx);
        /* Sign bias for alternate frame */
        p->refFrameSignBias[1] = vp8hwdDecodeBool128(bit_ctx);
        /* Refresh entropy probs */
        p->refreshEntropyProbs = vp8hwdDecodeBool128(bit_ctx);
        /* Refresh last */
        p->refreshLast = vp8hwdDecodeBool128(bit_ctx);
    }
```

### 9.8 Refresh Last Frame Buffer  

这个bit只有在非关键帧的时候有，关键帧的时候，这个bit是被覆盖的。实现就是上面的最后一行：p->refreshLast = vp8hwdDecodeBool128(bit_ctx);

![image-20200629153820637](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202107.png)

### 9.9 DCT Coefficient Probability Update

![image-20200629154046198](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202115.png)

这个描述说具体介绍在第十三章，介绍这个是如何更新的，且看第十三章

![image-20200629154622028](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202125.png)

实现如下: 

```C
vp8hwdDecodeCoeffUpdate(p);

static void vp8hwdDecodeCoeffUpdate(VP8DParserContext_t *p)
{
    RK_U32 i, j, k, l;
    FUN_T("FUN_IN");
    for ( i = 0; i < 4; i++ ) {
        for ( j = 0; j < 8; j++ ) {
            for ( k = 0; k < 3; k++ ) {
                for ( l = 0; l < 11; l++ ) {
                    if (vp8hwdDecodeBool(&p->bitstr,
                                         CoeffUpdateProbs[i][j][k][l]))
                        p->entropy.probCoeffs[i][j][k][l] =
                            vp8hwdReadBits(&p->bitstr, 8);
                }
            }
        }
    }
    FUN_T("FUN_OUT");
}
```



### 9.10 Remaining Frame Header Data (non-Key Frame)

![image-20200629154115595](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202132.png)



### 9.11 Remaining Frame Header Data (Key Frame)  

![image-20200629154245272](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202141.png)

9.10和9.11的实现均如下：

```C
p->coeffSkipMode =  vp8hwdDecodeBool128(bit_ctx);
if (!p->keyFrame) {
    RK_U32  mvProbs;
    if (p->coeffSkipMode)
        p->probMbSkipFalse = vp8hwdReadBits(bit_ctx, 8);
    p->probIntra = vp8hwdReadBits(bit_ctx, 8);
    p->probRefLast = vp8hwdReadBits(bit_ctx, 8);
    p->probRefGolden = vp8hwdReadBits(bit_ctx, 8);
    if (vp8hwdDecodeBool128(bit_ctx)) {
        for (i = 0; i < 4; i++)
            p->entropy.probLuma16x16PredMode[i] = vp8hwdReadBits(bit_ctx, 8);
    }
    if (vp8hwdDecodeBool128(bit_ctx)) {
        for (i = 0; i < 3; i++)
            p->entropy.probChromaPredMode[i] = vp8hwdReadBits(bit_ctx, 8);
    }
    mvProbs = VP8_MV_PROBS_PER_COMPONENT;
    for ( i = 0 ; i < 2 ; ++i ) {
        for ( j = 0 ; j < (RK_S32)mvProbs ; ++j ) {
            if (vp8hwdDecodeBool(bit_ctx, MvUpdateProbs[i][j]) == 1) {
                tmp = vp8hwdReadBits(bit_ctx, 7);
                if ( tmp )
                    tmp = tmp << 1;
                else
                    tmp = 1;
                p->entropy.probMvContext[i][j] = tmp;
            }
        }
    }
} else {
    if (p->coeffSkipMode)
        p->probMbSkipFalse = vp8hwdReadBits(bit_ctx, 8);
}
```