这篇note主要是记录下看vp8标准文档的一个记录。

![image-20200806163132870](https://cdn.jsdelivr.net/gh//fumasterlin/cloudimg/notes_img/20200806163140.png)

- 一个编码帧由3个或3个以上的块组成

  1. 首先会由一个uncomppress的数据
     - 关键帧是 10byte， 非关键字是 3byte

  2. 接着是两个及以上的块组成，vp8里叫做partition