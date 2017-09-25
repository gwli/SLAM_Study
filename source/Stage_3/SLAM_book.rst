SLAM 14讲
*********


ch4 使用李群与李代数
====================


由旋转矩阵是李群，然后通过李代数的映射来计算旋转矩阵。相似变换也是李群。

.. figure:: /Stage_3/SLAM_book/ch4_useSophus.png

   useSophus 的profiling分析

#. 由于helloworld级的代码，大部分CPU时间都花在了ld-2.23.0
   再查其callstack大部分时间花在了 :command:`dl_lookup_symbol_x` 这个函数。 
   dl_lookup_symbols_x 是glib C runtime 库，用来查动态链接库中的符号查询。
   解决见 `stackowerflow <https://stackoverflow.com/questions/11768919/what-is-dl-lookup-symbol-x-c-profiling>`_

#. 真正profiling算法本身的性能时，可能就要代码中加入一些mark标记，这样才能准确的profiling.
   或者在profiling时加入一定等待时间。



.. figure:: /Stage_3/SLAM_book/ch4_useSophus_timeline.png

   useSophus 的timeline 

#. 看起来基本是单线程也没有跑满。


CH5相机与图像
=============


.. figure:: /Stage_3/SLAM_book/ch5_imageBasics.png
   
   imageBasics 

.. figure:: /Stage_3/SLAM_book/ch5_imageBasics_timeline.png

   imageBasics timeline

从上面的图中可以看到，一个单线程的进程，却发生了几次 thread context的切换。并且每一次的
切换也都耗费了两分钟。

当不是很规范的时候，例如一个大函数搞定一切，如果想进一步了解性能原因，那就要用到
nvtx来标记了，或者支持源码级的优化，也就是每一个行指令的执行的次数，或者利用
颜色标记出来，例如颜色越深的地方，也就是执行次数最多的地方。当然就需要instruction
级别的profiling了。

同时利用脚本来执行这些profiling最后集中查看会更加方便。 甚至一个整套的profiling测试
修改源码，编译，执行profiling测试，并且根据profiling的数据生成测试分析报告图表。


同时在一个机器编译的binary,如何快速复制其他device上也能跑呢，并且其又依赖大量的库。
一个个机器去装这些依赖库也挺麻烦的。 一个快速的方法那就是
直接用ldd 把 每一个binary 的依赖的 so 打包。然后再加上LD_PRELOAD来加载，这样就可以加快部署的进程。
例外一种还需要多线程的操作，可能需要同时操作两条线。例如

#. 一个线程要操作 binary本身
#. 另一个线程要操作 profiling等等。

位姿记录的形式是平移向量旋转四元数: :math:`[x,y,z,q_x,q_y,q_z,q_w]`

而生成地图的格式 `PCL pcd file format <http://pointclouds.org/documentation/tutorials/pcd_file_format.php>`_


ch6 非线优化
============

Ceres 
-----

g2o
----


ch7 

ch9
===

工程会报错 opencv/viz.hpp没有
-----------------------------


.. code-block:: bash

   dpkg -L  opencv //可以查看这个库到底安装了哪些文件
   apt install -y libvtk5-dev
   cmake -DWITH_VTK=On <path to your opence srouce> 
   make 
   make install  //update the include path
