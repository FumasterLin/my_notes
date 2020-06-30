# H264 NALU详解

(转载自https://www.jianshu.com/p/a2dc69c8bf70）

##### 1. H264码流格式

不过大道始于脚下，我们还是先从头介绍一下，h264的两种码流格式，它们分别为：字节流格式和RTP包格式。

（1）字节流格式：这是在h264官方协议文档中规定的格式，处于文档附录B（Annex-B Byte stream format）中。所以它也成为了大多数编码器，默认的输出格式。它的基本数据单位为NAL单元，也即NALU。为了从字节流中提取出NALU，协议规定，在每个NALU的前面加上起始码：0x000001或0x00000001（0x代表十六进制） 。

（2）RTP包格式：这种格式并没有在h264中规定，那为什么还要介绍它呢？是因为在h264的官方参考软件JM里，有这种封装格式的实现。在这种格式中，NALU并不需要起始码Start_Code来进行识别，而是在NALU开始的若干字节（1，2，4字节），代表NALU的长度。

显而易见，我们通常所指，以及接下来要研究的，是h264的字节流格式。由于它没有经过传输协议封装，所以也可以称之为裸流。比如我们打开一个，经编码器编码存于本地后缀为.h264文件，里面的数据即为h264裸流。

而我们接下来的研究方向，就从已经打开了一个本地的.h264文件，然后对里面的h264裸流，按照字节流格式进行分析开始。所以拿到码流的第一刻，我们需要知道，如何从中提取出NALU。

##### 2. 起始码与NALU

通过上面我们已经知道：

> H264比特流 = Start_Code_Prefix + NALU +  Start_Code_Prefix + NALU + …

只要我们从码流中，找到一个一个的起始码，那么位于起始码之间的数据，即为NALU。所以拿到码流，我们需要先从头开始，找到起始码0x000001或0x00000001，找到Start_Code_Prefix之后，从它之后的下一个字节开始，就是NALU的部分。

这部分的实现过程描述在H264官方文档附录B中，已经下载的同学可以查看B.1.1节：

![img](https:////upload-images.jianshu.io/upload_images/4272749-2d08c2310aa2b296?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

B.1.1 字节流NAL单元语法

##### 3. NALU

看到这一小节时，我们已经有能力根据附录B的内容，从h264码流中找出NALU，所以是时候来看一下，h264码流结构的组成了：

![img](https:////upload-images.jianshu.io/upload_images/4272749-ffe0ae6f39d657a3?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

NALU构成H264码流结构

这就是NALU在H264码流中的构成了，由上图我们也知道：

##### NALU = NALU Header + RBSP

这就是接下来我们要干的，分析NALU Header 和 RBSP，为了对NALU有个宏观的认识，我们先来看一下，NALU有哪些句法元素构成，这位于h264文档的7.3.1节：

![img](https:////upload-images.jianshu.io/upload_images/4272749-98c91ce6b17d80f9?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

7.3.1节 NALU句法元素构成

可以看到，整个NALU语法元素分为三部分：（1）NALU Header、（3）RBSP、（2）1和3之间的部分。

其中第2部分，是近期的h264文档才更新的，所以我特意查看了JM、x264、FFmpeg等主流编解码器，这部分是还没有实现的，所以我们可以不必理睬。而且细心的同学会发现，只有当nal_unit_type等于14、20、21时，才会进入第二部分。

所以接下来呢，我们就重点介绍NALU Header和RBSP。

##### 3.1 NALU Header

通过上面我们也可以看到，NALU Header由三个句法元素组成，分别为：forbidden_zero_bit、nal_ref_idc和nal_unit_type，它们总共占据一个字节，也就是说，NALU Header，在整个NALU中，占据一个字节。

而且forbidden_zero_bit的值对应1个bit，nal_ref_idc的值对应2个bit，nal_unit_type的值对应5个bit，加起来刚好一个字节。

正如在上一篇（链接）中所介绍的，知道了句法元素，我们就来分别看看它的语义：

##### 3.1.1 forbidden_zero_bit

h264文档规定，这个值应该为0，当它不为0时，表示网络传输过程中，当前NALU中可能存在错误，解码器可以考虑不对这个NALU进行解码。

##### 3.1.2 nal_ref_idc

取值0~3，代表当前这个NALU的重要性，取值越大，代表当前NALU越重要，就需要优先被保护。尤其是当前NALU为图像参数集、序列参数集或IDR图像时，或者为参考图像条带（片/Slice），或者为参考图像的条带数据分割时，nal_ref_idc值肯定不为0。

而当NALU 类型，nal_unit_type为6、9、10、11、或12时，nal_ref_idc都为0。

【注】IDR帧，即：即时解码刷新图像，它是一个序列的第一个图像，H.264引入IDR图像是为了解码的重新同步。当解码器解码到IDR图像时，立即将参考帧队列清空，将已解码的数据全部输出或抛弃，重新查找参数集，开始一个新的序列。这样一来，如果前一个序列发生重大错误，在这里就可以获得重新同步。

所以IDR图像之后的图像，永远不会引用IDR图像之前的图像来解码。并且IDR图像一定是I图像，而I图像不一定是IDR图像（H264里没有图像层，图像可以理解为帧、片或宏块）。

##### 3.1.3 nal_unit_type

顾名思义，这个应该是最好理解的了，它表示NALU Header后面的RBSP的数据结构的类型。下图为nal_unit_type所有可能的取值，和对应的语义，它处于h264文档7.4.1节：

​																										表  nal_uint_type 语义

| nal_unit_type | NAL类型                  | C       |
| ------------- | ------------------------ | ------- |
| 0             | 未使用                   |         |
| 1             | 不分区、非 IDR 图像的片  | 2, 3, 4 |
| 2             | 片分区 A                 | 2       |
| 3             | 片分区 B                 | 3       |
| 4             | 片分区 C                 | 4       |
| 5             | IDR 图像中的片           | 2, 3    |
| 6             | 补充增强信息单元（ SEI） | 5       |
| 7             | 序列参数集               | 0       |
| 8             | 图像参数集               | 1       |
| 9             | 分界符                   | 6       |
| 10            | 序列结束                 | 7       |
| 11            | 码流结束                 | 8       |
| 12            | 填充                     | 9       |
| 13..23        | 保留                     |         |
| 24..31        | 未使用                   |         |

可以看到，nal_unit_type的值为1-5时，表示RBSP里面包含的数据为条带（片/Slice）数据，所以值为1-5的NALU统称为VCL（视像编码层）单元，其他的NALU则称为非VCL NAL单元。

当nal_unit_type为7时，代表当前NALU为序列参数集，为8时为图像参数集。这也是我们打开.h264文件后，遇到的前两个NALU，它们位于码流的最前面。

而且当nal_unit_type为14-31时，我们可以不用理睬，目前几乎用不到。



我们从h264裸流中，提取出一个个的NALU，并且解析出NALU的第一个字节：NALU Header。下面我们就从NALU Header的下一个字节开始，分析NALU剩余的数据部分，也即NALU的主体部分。

NALU的主体涉及到三个重要的名词，分别为EBSP、RBSP和SODB。其中EBSP完全等价于NALU主体，而且它们三个的结构关系为：

##### EBSP包含RBSP，RBSP包含SODB。

其中SODB就是最原始的编码数据。



##### 4. EBSP和RBSP

上面我们说，NALU的组成部分为：

##### NALU = NALU Header + RBSP

其实严格来说，这个等式是不成立的，因为RBSP并不等于NALU刨去NALU Header。严格来说，NALU的组成部分应为：

##### NALU = NALU Header + EBSP

其中的EBSP为**扩展字节序列载荷（Encapsulated Byte Sequence Payload）**，而RBSP为**原始字节序列载荷（Raw Byte Sequence Payload）**。那为什么我们上篇中，没有使用2式而使用了1式呢？那是因为，在h264的文档中，并没有EBSP这一名词出现，但是在h264的官方参考软件JM里，却使用了EBSP。

而且在我们下面的分析中，我们会看到，使用EBSP是很易于理解的。

##### 4.1 防止竞争

EBSP相较于RBSP，多了防止竞争的一个字节：0x03。

我们知道，NALU的起始码为0x000001或0x00000001，同时H264规定，当检测到0x000000时，也可以表示当前NALU的结束。那这样就会产生一个问题，就是如果在NALU的内部，出现了0x000001或0x000000时该怎么办？

所以H264就提出了“防止竞争”这样一种机制，当编码器编码完一个NAL时，应该检测NALU内部，是否出现如下左侧的四个序列。当检测到它们存在时，编码器就在最后一个字节前，插入一个新的字节：0x03。

![image-20200615142924054](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629195607.png)

防止竞争插入0x03

图中0x000000和0x000001前面介绍了，0x000002是作为保留使用，而0x000003，则是为了防止NALU内部，原本就有序列为0x000003这样的数据。

这样一来，当我们拿到EBSP时，就需要检测EBSP内是否有序列：0x000003，如果有，则去掉其中的0x03。这样一来，我们就能得到原始字节序列载荷：RBSP。

##### 5. RBSP和SODB

得到RBSP之后，我们迫切想做的，就是从RBSP中，提取出原始编码数据SODB（String Of Data Bits）。这就涉及到RBSP与SODB的关系：

##### RBSP = SODB + RBSP尾部

而且RBSP的尾部，在规定中有两种，我们分别介绍。

##### 5.1 RBSP尾部

其中大多数类型的NALU，使用这种尾部。

![img](https:////upload-images.jianshu.io/upload_images/4272749-4256669759e63bd6?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

RBSP尾部语法 （文档7.3.2.11）

其中：

rbsp_stop_one_bit 占1个比特位，值为1

rbsp_alignment_zero_bit  值为0，目的是为了进行字节对齐，占据若干比特位

所以RBSP就等于，SODB在它的最后一个字节的最后一个比特后，紧跟值为1的1个比特，然后增加若干比特的0，以补齐这个字节。

##### 5.2 条带RBSP尾部

另一种尾部，就是当NALU类型为条带时，也即nal_unit_type等于1~5时，这时RBSP使用下面这种尾部：

![img](https:////upload-images.jianshu.io/upload_images/4272749-23c82d0d82076f69?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

条带RBSP尾部语法（7.3.2.10）

可以看到，rbsp_slice_trailing_bits()默认情况下，就是2.1介绍的第一种尾部。只是当entropy_coding_mode_flag值为1，也即当前采用的熵编码为CABAC，而且more_rbsp_trailing_data()返回为true，也即RBSP中有更多数据时，添加一个或多个0x0000。

所以我们拿到RBSP，只需要按照上述语法，去掉RBSP的尾部，就可以得到SODB。然后就可以对照对应类型的NALU的句法，解析出语法元素的值。

总结上篇和这篇，H264的码流结构如下：

![img](https:////upload-images.jianshu.io/upload_images/4272749-1ef27074f7094ae1?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

##### 6. H264句法元素解析流程

而当我们拿到RBSP或SODB之后，就可以对照各类型的NALU，去解析它们的语法元素，进而再根据语法元素，重建图像。其中解析语法元素的框图如下：

![img](https:////upload-images.jianshu.io/upload_images/4272749-6a9e2282d504fa34?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



由图可见，解析NALU的各个句法元素并不难，只要根据h264文档对应章节的句法，并配合相应的编解码算法解析即可。