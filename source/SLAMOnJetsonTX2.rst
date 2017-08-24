SLAM on Jetson TX2
==================

.. graphviz::
   
   digraph G {
       node [shape=box];
       Camera ->"Jetson TX2"-> "V4L2Engines"->CUDA->TensorRT->GL;
       "Decodes from video" ->"run deep learning"->"do post processingwithCUDAandGraphics";
   }


#. Install CUDA

   .. code-block:: bash

      sudo dpkg -i cuda-repo-ubuntu1604-8-0-local-r381_8.0.84-1_amd64.deb
      sudo apt-get 
      sudo apt-get install -y --allow-unauthenticated update
      sudo apt-get install -y cuda 
      sudo dpkg --add-architecture arm64
      sudo apt-get --allow-unauthenticated update
      sudo apt-get install -y --allow-unauthenticated cuda-cross-aarch64

#. libopencv4tegra-dev

   .. code-block:: bash

      sudo apt-get install -y cuda-cross-aarch64
