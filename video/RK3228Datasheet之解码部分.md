### 简述

本文主要记录一下怎么datasheet里看decoder部分的内容，以rk3228为例。

decoder部分我们主要看下面这份pdf文件：

```
Rockchip RK3228 TRM V0.1 20151016-Part3 Graphic and Multi-media.pdf
```

我们主要看其中的chapter 4 Multi-format Video Decoder部分

![image-20200622170745934](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202152.png)

chapter 4的内容展开如下，这里面的内容我们一般都要看的。

![image-20200622170930655](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202159.png)

### Overview

Overview里主要介绍了支持的一些codec。

比如支持 H264解码：

- 可以支持到4K（4096x2304）@30fps。
- 支持一些错误时候的中断
- 支持slice by slice的解码
- 等等

![image-20200622171215154](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202212.png)

### Block Diagram

这里是介绍了下cpu和mem 模块 和解码器的交互。

cpu设置一些寄存器去对解码器做一些控制的操作，是通过AHB Bus，解码器和mem通过AXI Bus来做一些内存的交互。

![image-20200622171536562](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202213.png)

Video frame format  

这一章是介绍一些硬件所支持的图片格式，以及对具体的图像格式做一些介绍。

![image-20200622171830807](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202214.png)

### Function Description  

这一章节看目录就知道讲了几个内容，其中着重看：

- HEVC/H264/VP9工作模式
- HEVC/H264/VP9输入数据格式的要求
- HEVC/H264/VP9错误处理

![image-20200622175219367](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202216.png)

#### working mode

以H264为例，这里列举了三种工作模式：

- RLC mode
- RLC write mode
- normal mode

![image-20200622181058572](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202217.png)

#### input data

这里规定了一些input data的格式要求，字节对齐要求。

以H264为例：

sps和pps的输入要求，在code里需要按照如下的要求填充spspps的buffer，然后将buffer的地址配置给寄存器

![image-20200622193156012](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202218.png)

![image-20200622193219532](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202219.png)

rps输入要求，在code里需要按照如下的要求填充rps的buffer，然后将buffer的地址配置给寄存器

![image-20200622193413084](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629202220.png)

#### error process

以H264为例，错误信息可分为两种：

- normal error

When swreg1_int[19] is configured as 1’d1, the hardware will wait the end signal of deblocking and then reset itself. And for normal error, the error info will be put on the address swreg4_strm_rlc_base. Specific data are listed in the following table:  

- logic error

当出现错误信息的时候，硬件会stop并且reset。

> 当出现错误的时候寄存器swreg76_h264_errorinfo_num[13:0]将会记录the number of slices in frame,

> swreg76_h264_errorinfo_num[15] will be set to1,

> swreg76_h264_errorinfo_num[29:16] will record the number of error slices in frame.

### Register Description  

这个章节更具体列出了相关的寄存器介绍



### H264 Configuration Flow

1. Prepare the data in the DDR, for normal mode, we should prepare bitstream, tbl, pps and rps.

2. Set the h264 general system configuration in RKVDEC.swreg2, such as working mode, in/out endian.

3. Set the picture parameters with RKVDEC.swreg3.

4. Set the input and output data base address and H264 reference configuration with RKVDEC.swreg4~RKVDEC.swreg43.

5. If stream error detection is desired, set the swreg77_h264_error_e and swreg44_strmd_error_en to enable the corresponding error detection.

6. If prefech function is desired, set the prefetch common registers and clear its TLB. Pay attention, there contains two caches, which are for Y channel and UV channel.

7. If MMU function is desires, set the MMU common registers and clear its TLB. Pay attention, there contains two MMUs, which are for read channel and write channel.

8. Set the interrupt configuration and start the decoder with RKVDEC.swreg1.

9. Wait for the frame interrupt, and then get the processed results in the target DDR. There may be decoded frame, error_info, cur frame colmv output.
   When the stream mode is not frame by frame mode,we also wait buf empty,and then send the next pack,repeat it until sw_dec_rdy_sta.

10. Clear all the interrupts, and repeat Process2~Process9 to start a new frame decoding if 
    the decoding is not finished yet.  