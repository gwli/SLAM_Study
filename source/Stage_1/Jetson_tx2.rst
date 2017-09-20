Jetson-TX2
**********

Jeton-TX2 的开发包，包含了 tensorRT,cuda,cudnn,opencv,以及visionworks 以及Multimedia API等等。 以及相关的开发工具，CPU的proiler, NSight Eclipse, Tegra Graphic profiler 等等。并且本身是全版的ubuntu 16.04. 特别适合做二次开发。

JetPack 的目录结构
==================

.. literalinclude:: /Stage_1/Jetpack_directory.txt
   :language: bash
   :emphasize-lines: 12,22 
   
   


如何快速定制target的刷机Image
=============================


#. 从官网下载最新的 JetPack_ 最新版本为3.1.
#. 选择相关的包进行安装并刷机。

   .. note::
      
      进入recover 模式， 组合键顺序:   :kbd:`Rec` -> :kbd:`Rec+Reset` -> :kbd:`Rec` 

#.  在device 上安装各种额外的包。你可以用VNC 或者ssh 去连接device.
    
    .. code-block:: bash
     
       sudo apt-get intall -y git cmake clang

#. 备份整个device的rootfs.

    .. code-block:: bash
      
       sudo tar -cvpz --one-file-system / |ssh <yourlocalhost> "(cat >ssh_jetson_tx2_rootfs.tgz)"

#. 在host上解压到jetpack的解压目录
 
   .. code-block:: bash
       
      sudo tar -xvpzf /path/to/ssh_jetson_tx2_rootfs.tar.gz -C <rootfs folder in host> --numeric-owner"


#. 重新刷机生成一个新的刷机包

   .. code-block:: bash
      
      cd <Jetpackppath>/64_TX2/Linux_for_Tegra_tx2
      ./flash_os jetson-tx2 mmcblk0p1

   .. note::
      
      刷机前看一下device是否在 *recovery mode*
      用命令 :command:`lsusb | grep "nvidia"` 来查看

#. 等刷机完成后，在 :file:`Linux_for_Tegra_tx2/bootloader` 下就生成了新的刷机包了。并且以后刷机直接用
   :command:`cd Linux_for_Tegra_tx2/bootloader && sh bootloader` 。基本十几分钟就可以刷好了。














参考
====
.. _JetPack: https://developer.nvidia.com/embedded/jetpack
