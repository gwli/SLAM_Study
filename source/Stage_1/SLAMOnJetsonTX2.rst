******************
SLAM on Jetson TX2
******************


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


TX2 的基本库
============

计算有CUDA,图形 GL，图像OPENCV，以及计算机视觉有Visionworks。DL可以用tensorrt来推理。

标准流可以用vision works,非标准流可以用 tegra-multimedia api.

基本的视频分析
==============


Tegra_multimedia sample
=======================

#. backend
