### 问题现象：

播放一段webm的视频，大概十几秒的时候画面会卡住。

### 问题分析

拿到问题视频文件，本地复现。

1. 打开mpp_dec的log，抓取log

   从log里看到这里确实卡住了，最后的log是

   Line 21931: 01-01 15:44:03.187   195  4408 I mpp_dec : detail: check output index pass

   正常的情况下，往下还会有如下的log紧随而来，那下面这个log没有打出来，log里也没有看到线程有crash的迹象，说明这之间return了，才导致没有继续往下走。

   mpp_dec : detail: check frame group count pass

   ```
   	Line 21907: 01-01 15:44:03.152   195  4408 I mpp_dec : detail: one task ready
   	Line 21909: 01-01 15:44:03.154   195  4408 I mpp_dec : detail: check prev task pass
   	Line 21911: 01-01 15:44:03.154   195  4408 I mpp_dec : detail: check mframes pass
   	Line 21913: 01-01 15:44:03.154   195  4408 I mpp_dec : detail: check output index pass
   	Line 21915: 01-01 15:44:03.184   195  4408 I mpp_dec : detail: check prev task pass
   	Line 21917: 01-01 15:44:03.185   195  4408 I mpp_dec : detail: check mframes pass
   	Line 21919: 01-01 15:44:03.185   195  4408 I mpp_dec : detail: check output index pass
   	Line 21921: 01-01 15:44:03.185   195  4408 I mpp_dec : detail: check frame group count pass
   	Line 21923: 01-01 15:44:03.185   195  4408 I mpp_dec : detail: check output buffer 0xb8c17328
   	Line 21925: 01-01 15:44:03.185   195  4408 I mpp_dec : detail: one task ready
   	Line 21927: 01-01 15:44:03.187   195  4408 I mpp_dec : detail: check prev task pass
   	Line 21929: 01-01 15:44:03.187   195  4408 I mpp_dec : detail: check mframes pass
   	Line 21931: 01-01 15:44:03.187   195  4408 I mpp_dec : detail: check output index pass
   ```

   所以我们进入code里去看打印这两行log之间做了什么事情？根据如下的code，我们确实看到这两行log之间确实有两处可能return的点，为了进一步确认问题点，我们在两个return的点前面各加一行log来定位。

   ```C
       dec_dbg_detail("detail: check output index pass\n");
   
       /*
        * 9. parse local task and slot to check whether new buffer or info change is needed.
        *
        * detect info change from frame slot
        */
       if (mpp_buf_slot_is_changed(frame_slots)) {
           if (!task->status.info_task_gen_rdy) {
               RK_U32 eos = task_dec->flags.eos;
   
               // NOTE: info change should not go with eos flag
               task_dec->flags.info_change = 1;
               task_dec->flags.eos = 0;
               mpp_dec_put_task(mpp, task);
               task_dec->flags.eos = eos;
   
               task->status.info_task_gen_rdy = 1;
               return MPP_ERR_STREAM;
           }
       }
       dec_dbg_detail("dd 1\n");
       task->wait.info_change = mpp_buf_slot_is_changed(frame_slots);
       if (task->wait.info_change) {
           return MPP_ERR_STREAM;
       } else {
           task->status.info_task_gen_rdy = 0;
           task_dec->flags.info_change = 0;
           // NOTE: check the task must be ready
           mpp_assert(task->hnd);
       }
       dec_dbg_detail("dd 2\n");
       /* 10. whether the frame buffer group is internal or external */
       if (NULL == mpp->mFrameGroup) {
           mpp_log("mpp_dec use internal frame buffer group\n");
           mpp_buffer_group_get_internal(&mpp->mFrameGroup, MPP_BUFFER_TYPE_ION);
       }
   
       /* 10.1 look for a unused hardware buffer for output */
       if (mpp->mFrameGroup) {
           RK_S32 unused = mpp_buffer_group_unused(mpp->mFrameGroup);
           MppBufferGroupImpl *p = (MppBufferGroupImpl *)(mpp->mFrameGroup);
   
           // NOTE: When dec post-process is enabled reserve 2 buffer for it.
           task->wait.dec_pic_unusd = (dec->vproc) ? (unused < 3) : (unused < 1);
           mpp_log("mFrameGroup,limit_count:%d, count_unused:%d, count_used:%d,unused:%d, wait:%d", p->limit_count, p->count_unused, p->count_used,unused, task->wait.dec_pic_unusd);
           if (task->wait.dec_pic_unusd)
               return MPP_ERR_BUFFER_FULL;
       }
       dec_dbg_detail("detail: check frame group count pass\n");
   ```

2. 加了log后我们进一步复现抓log

   ```C
   	Line 20914: 01-01 15:57:06.817   195  1318 I mpp_dec : detail: check prev task pass
   	Line 20916: 01-01 15:57:06.817   195  1318 I mpp_dec : detail: check mframes pass
   	Line 20918: 01-01 15:57:06.817   195  1318 I mpp_dec : detail: check output index pass
   	Line 20920: 01-01 15:57:06.817   195  1318 I mpp_dec : 22 1
   	Line 20922: 01-01 15:57:06.817   195  1318 I mpp_dec : dd 2
   	Line 20924: 01-01 15:57:06.817   195  1318 I mpp_dec : detail: check frame group count pass
   	Line 20926: 01-01 15:57:06.817   195  1318 I mpp_dec : detail: check output buffer 0xb8831178
   	Line 20928: 01-01 15:57:06.818   195  1318 I mpp_dec : detail: one task ready
   	Line 20930: 01-01 15:57:06.820   195  1318 I mpp_dec : detail: check prev task pass
   	Line 20932: 01-01 15:57:06.820   195  1318 I mpp_dec : detail: check mframes pass
   	Line 20934: 01-01 15:57:06.820   195  1318 I mpp_dec : detail: check output index pass
   	Line 20936: 01-01 15:57:06.820   195  1318 I mpp_dec : 22 1
   	Line 20938: 01-01 15:57:06.821   195  1318 I mpp_dec : dd 2
   ```

   可以定位到是第二个return的地方挡掉了，我们看具体的code，可以看出来这里是确认mFrameGroup里的unused buffer数量，如果小于一定数目就直接return了

   ```C
       dec_dbg_detail("dd 2\n");
   ..
       /* 10.1 look for a unused hardware buffer for output */
       if (mpp->mFrameGroup) {
           RK_S32 unused = mpp_buffer_group_unused(mpp->mFrameGroup);
           MppBufferGroupImpl *p = (MppBufferGroupImpl *)(mpp->mFrameGroup);
   
           // NOTE: When dec post-process is enabled reserve 2 buffer for it.
           task->wait.dec_pic_unusd = (dec->vproc) ? (unused < 3) : (unused < 1);
           mpp_log("mFrameGroup,limit_count:%d, count_unused:%d, count_used:%d,unused:%d, wait:%d", p->limit_count, p->count_unused, p->count_used,unused, task->wait.dec_pic_unusd);//dd
           if (task->wait.dec_pic_unusd)
               return MPP_ERR_BUFFER_FULL;
       }
       dec_dbg_detail("detail: check frame group count pass\n");
   ```

   另外在上面也加了一句log，mFrameGroup里的buffer变化情况（注释//dd处）

