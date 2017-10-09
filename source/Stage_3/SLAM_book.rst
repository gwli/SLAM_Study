SLAM 14讲
*********

目标
====

#. 用Nsight Tegra 来分析每一个例子瓶颈
#. 用LLVM 生成callgraph， 如何查看其编译的参数
#. 找出各种优化方法
#. 并且写出验证代码
#. 写出自动优化的框架
#. 生成各种report 
#. 实现debug过程的可视化,可以通过扩展gdb,以及底层的api为一些动态的数据。

ch3 visualizeGeometry
======================

.. image::  /Stage_3/SLAM_book/ch2_visualizeGeomtry.png

用了一个 Pangolin的GUI库，

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

用法用核心就是 定义loss函数。 然后用把数据扔给solver就行了。

看一个hello world就明白了。

.. math::

   Loss = min \sum | y^1 -f(x)|^2


.. code-block:: cpp

   struct CostFunctor {
   template <typename T>
   bool operator()(const T* const x, T* residual) const {
     residual[0] = T(10.0) - x[0];
     return true;
   }
   };

   int main(int argc, char** argv) {
     google::InitGoogleLogging(argv[0]);
   
     // The variable to solve for with its initial value.
     double initial_x = 5.0;
     double x = initial_x;
   
     // Build the problem.
     Problem problem;
   
     // Set up the only cost function (also known as residual). This uses
     // auto-differentiation to obtain the derivative (jacobian).
     CostFunction* cost_function =
         new AutoDiffCostFunction<CostFunctor, 1, 1>(new CostFunctor);
     problem.AddResidualBlock(cost_function, NULL, &x);
   
     // Run the solver!
     Solver::Options options;
     options.linear_solver_type = ceres::DENSE_QR;
     options.minimizer_progress_to_stdout = true;
     Solver::Summary summary;
     Solve(options, &problem, &summary);
   
     std::cout << summary.BriefReport() << "\n";
     std::cout << "x : " << initial_x
               << " -> " << x << "\n";
     return 0;
   }

g2o
----

图优化的目标就是把用优化问题变成图优。

优化问题 :math:`\min\limits_{x} F(x)` 三个基本因素:

#. 目标函数
#. 优化变量
#. 优化约束


最基本的图优化就是用图模型来表达一个非线性最小二乘的优化问题。

图优化的原理
在图中，以顶点表示优化变量，以边表示观测方程或者边为误差项。 我们目标最短路径
或者全体权值最小。

在图中，我们去掉孤立顶点或化先优化边数较多的顶点。

.. math:: 
   
   \min\limits_{x} \sum\limits_{k = 1}^n {{e_k}{{\left( {{x_k},{z_k}} \right)}^T}{\Omega _k}{e_k}\left( {{x_k},{z_k}} \right)} 



与ceres 类似，这个是一个通用优化框架，你需要继承或定义问题本身的基本模型就可以了。
例如g2o就是要定义顶点类与边类的如何更新与计算。 把一堆的顶点与边扔进去。

ch7 VO
======

feature_exection
----------------

特征点的提取与描述子两部分组成。
特征点的提取计算与以及有准备性。

.. image:: /Stage_3/SLAM_book/ch7_feature_extraction.png
.. image:: /Stage_3/SLAM_book/ch7_feature_extraction_ORB.png
.. image:: /Stage_3/SLAM_book/ch7_feature_extraction_Analysis_summary.png
.. image:: /Stage_3/SLAM_book/ch7_feature_extraction_timeline.png

.. csv-table:: 

   :header: "method",comments

    SIFT, 
    FAST,
    ORB, Oriented FAST and　Rotateed BRIEF
   
1000 个字， ORB 15.3ms, SURF,217.3ms, SIFT 5228.7ms. 

3D to 3D 的位置估计
--------------------

也就是从自己观测的3D点，来计算出自身的运动方程

.. image::  /Stage_3/SLAM_book/ch7_pose_estimation_3d3d.png

这个基本上都还是单线程。耗时比较除了do_lookup_x之外，那就是cv::FAST函数了。




ch8 V0
======

LKFlow
------

.. image:: /Stage_3/SLAM_book/ch8_LK_Analysis.png
.. image:: /Stage_3/SLAM_book/ch8_LK_timeline.png
.. image:: /Stage_3/SLAM_book/ch8_LK.png

DirectSemiDense and DirectSparse
--------------------------------

Buserror wait for debug

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


ch10
====

g20_bundle
==========

.. image:: /Stage_3/SLAM_book/ch10_g2oBundle.png

g2o BuildSystem消耗的时间比较多


cere bundle
===========

.. image:: /Stage_3/SLAM_book/ch10_g2oBundle.png

ch11
=====

build error: gtsam module
