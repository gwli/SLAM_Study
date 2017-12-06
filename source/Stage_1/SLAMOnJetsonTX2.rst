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
