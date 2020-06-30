# POC计算

在 H.264 中， 图像序列号（ POC）主要用于标识图象的播放顺序，同时还用于在对帧间预测片解码时，标记参考图像的初始图像序号，表明下列情况下帧或场之间的图像序号差别：使用时间直接预测模式的运动矢量推算时； B 片中使用固有模式加权预测时；解码器一致性检测时。

其中对于每个编码帧有两个图像序列号，分别称为顶场序列号（ TopFieldOrderCnt）和底场序列号 （ BottomFieldOrderCnt） ； 对 于 每个 编 码 场 有一 个 图 像 序列 号 ， 对 于一 个 编 码 顶场 其 称 为TopFieldOrderCnt，对于编码底场，其称为 BottomFieldOrderCnt；对于每个编码场对有两个图像序列号， TopFieldOrderCnt 和 BottomFieldOrderCnt 分别用于标记该场对的顶场和底场。 TopFieldOrderCnt和 BottomFieldOrderCnt 分别指明了相应的顶场/底场相对于前一个 IDR 图像（或解码顺序中前一个包含 memory_management_control_operation=5 的参考图像）的第一个输出场的相对位置。

由于 H.264 使用了 B 帧预测，使得图象的解码顺序并不一定等于播放顺序，但它们之间存在一定的映射关系。 H.264 中一共定义了三种 POC 的编码方法，句法元素 pic_order_cnt_type 就是用来通 知 解 码 器 该 用 哪 种 方 法 来 计 算 POC, 下 面 我 们 称 之 为 POC 类 型 。 TopFieldOrderCnt 和BottomFieldOrderCnt 根据当前序列参数集中的语法元素 pic_order_cnt_type（ POC 类型）的值不同，按照如图 8.5 的计算流程获得。

如当前图像的语法元素 memory_management_control_operation 等于 5 时，在当前图像解码完成 后 ， 进 行 图 像 序 列 号 的 重 新 初 始 化 操 作 ， 即 将 顶 场 和 底 场 的 POC 均 减 去PicOrderCnt( CurrPic )。 CurrPic 标志当前图像。函数 PicOrderCnt( )的实现参见图 8.6。

![image-20200623161316561](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629195751.png)  

对于图像序列号 POC， H.264 规范规定在 H.264 码流中不会出现下列情况：

- 一个IDR帧的Min( TopFieldOrderCnt, BottomFieldOrderCnt )不等于0；
- 一个IDR顶场的TopFieldOrderCnt不等于0 ；
-  一个IDR底场的BottomFieldOrderCnt不等于0。

因此对于一个编码 IDR 帧的两个场， TopFieldOrderCnt 和 BottomFieldOrderCnt 中至少有一个等
于 0。

同 时 规 范 规 定 码 流 中 不 允 许 存 在 数 据 导 致 TopFieldOrderCnt ， BottomFieldOrderCnt ，PicOrderCntMsb ， 或 FrameNumOffset 超 出 -231 到 231-1 的 范 围 。 如 我 们 使 用DiffPicOrderCnt( picA, picB )表明两幅图像播放的时间间隔，即 DiffPicOrderCnt( picA, picB ) =PicOrderCnt( picA ) - PicOrderCnt( picB )，这样码流中的所有数据必须满足 DiffPicOrderCnt( picA,picB )的取值范围 -215 到 215 – 1。理由如下：假设 X 是当前图像， Y 和 Z 是同一序列中的另两个图像 ， Y 和 Z 被 认 为 是 从 X 起 始 的 相 同 输 出 顺 序 方 向 ， 当 DiffPicOrderCnt( X, Y ) 和DiffPicOrderCnt( X, Z ) 均为正或负时，其和不能超出-231 到 231-1 的范围。

根 据 当前 图像 性 质的 不同 ， 其使 用的 图 像序 列号 POC 会 有 所 区别 ，因 此 使用 函数PicOrderCnt( picX )标记图像 picX 的图像序列号，很多应用中 PicOrderCnt( X )正比于图像 X 的采样时间与 IDR 采样时间的差值，其函数如图 8.6 中定义： 

![image-20200623161628277](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629195943.png)

## pic_order_cnt_type  = 0 的POC计算方法

当 pic_order_cnt_type 等于 0 时，基于前一个参考图像（按照解码顺序）的 PicOrderCntMsb 计算当前图像的 TopFieldOrderCnt 和（或） BottomFieldOrderCnt。 首先计算变量 prevPicOrderCntMsb（前一个参考图像的 PicOrderCntMsb），然后计算当前图像的 PicOrderCntMsb，最后计算当前图像的TopFieldOrderCnt 和（或） BottomFieldOrderCnt。具体的计算流程参见图 8.7  

![image-20200623161918854](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629195956.png)

## pic_order_cnt_type  = 1 的POC计算方法

当 pic_order_cnt_type 等于 1 时， TopFieldOrderCnt 和（或） BottomFieldOrderCnt 按照流程图8.8 计算。主要基于前一图像（按照解码顺序）的 FrameNumOffset，来计算 TopFieldOrderCnt 和（或）BottomFieldOrderCnt。计算过程中涉及到两个变量 prevFrameNum 和 prevFrameNumOffset ，其中
prevFrameNum 是前一图像的 frame_num ，而对于 prevFrameNumOffset，如当前图像不是 IDR，而前一图像的 memory_management_control_operation 等于 5， prevFrameNumOffset 设为 0；否则，prevFrameNumOffset 设置等于前一图像的 FrameNumOffset  

![image-20200623161958740](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629200005.png)

## pic_order_cnt_type  = 2 的POC计算方法、

当 pic_order_cnt_type 等于 2 时， TopFieldOrderCnt 和（或） BottomFieldOrderCnt 的计算流程如图 8.9。 Picture order count 类型 2 不能用于包含连续非参考图像的序列中，且解码结果导致输出的顺序与解码顺序相同。流程图中 prevFrameNum 表示按照解码顺序的前一图像的 frame_num。如当前 图 像 不 是 IDR 图 像 ， 而 前 一 图 像 的 memory_management_control_operation 等 于 5 ，prevFrameNumOffset 设为 0；否则， prevFrameNumOffset 设置等于前一图像的 FrameNumOffset。  

![image-20200623162055399](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200629200019.png)