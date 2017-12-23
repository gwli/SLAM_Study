******************
SLAM on Jetson TX2
******************

TX2 的基本库
============

计算有CUDA,图形 GL，图像OPENCV，以及计算机视觉有Visionworks。DL可以用tensorrt来推理。

标准流可以用vision works,非标准流可以用 tegra-multimedia api.


视频分析
========

.. graphviz::
   
   digraph G {
       rankdir=LR;
       node [shape=box];
       Camera ->"Jetson TX2"-> "V4L2Engines"->CUDA->TensorRT->GL;
       "Decodes from video" ->"run deep learning"->"do post processingwithCUDAandGraphics";
   }

#. Capture video
#. Recognition objects
#. Take action in real-time
#. Visualize




Tegra_multimedia api sample
===========================

Manual : <L4T Multimedia API Reference> on https://developer.nvidia.com/embedded/downloads

库的组成
========

#. V4L2 API 用于各种视频的编解码与scaling等等。
#. libargus 用于图像处理,能直接处理lower level Camera信息。 具体流程可以查看http://on-demand.gputechconf.com/gtc/2016/webinar/getting-started-jetpack-camera-api.pdf
#. Buffer utilis ,buffer 的内存管理
#. NVDC-DRM 可以对于非 X11的轻量的级的显示管理系统，特别是适合一些嵌入的系统 
#. NVOSD on-Screen display.


backend
-------

`H.264 <https://zh.wikipedia.org/wiki/H.264/MPEG-4_AVC>`_  
3.1 支持了 GOOGLE 的VP9编码格式。

.. figure:: /Stage_1/image/backend.png

   from API reference manual

.. code-block:: bash

   $ ./backend 1 ../../data/Video/sample_outdoor_car_1080p_10fps.h264 H264 \
      --trt-deployfile ../../data/Model/GoogleNet_one_class/GoogleNet_modified_oneClass_halfHD.prototxt \
      --trt-modelfile ../../data/Model/GoogleNet_one_class/GoogleNet_modified_oneClass_halfHD.caffemodel \
      --trt-forcefp32 0 --trt-proc-interval 1 -fps 10

.. figure:: /Stage_1/image/backend_sp.png

   from Tegra System profiler

从主要进程来看其实现的功能，只需要选择一个线程上NVTX的标注或者各种低层API标注。以及查看只选择这个线程，
同时选择不同时段来看看下面的函数的统计，就知道其在干什么。
看NVTX的标注会比较准，而下面函数统计，则依赖于工具的准确性。


#. Dec_feed_loop thread 主要是V4L2 decode.
#. Cov0_capture :V4L2 block to pitch linear coversion
#. TensorRT Thread, 同步，copying data,物体的识别
#. Render Thread, 大部分时间等待数据来渲染

TimeLine 分析
-------------

#. 解码，识别，渲染的并行性不错
#. 大量的等待线程，可以建立一个  job system 来优化。
#. the long pole是不是可以更块，或者切成更小的并行stage 流水线来实现。

GPU

#. 填充timeline中大量的空隙，实现更高的利用率
#. 更有效的内存复用与传输机制 
#. 合并与优化 cuda kernel.
