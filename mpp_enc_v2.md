1. MPP_ENC_SET_HEADER_MODE

   可以设置header mode，是第一帧才有sps/pps，还是每个idr帧带sps/pps

   ```C
   typedef enum MppEncHeaderMode_t {
       /* default mode: attach vps/sps/pps only on first frame */
       MPP_ENC_HEADER_MODE_DEFAULT,
       /* IDR mode: attach vps/sps/pps on each IDR frame */
       MPP_ENC_HEADER_MODE_EACH_IDR,
       MPP_ENC_HEADER_MODE_BUTT,
   } MppEncHeaderMode;
   ```

   