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

Buserror wait for debug，这个是由 Trace/breakpoint Trap signal引起的，直接gdb时，gdb也直接发挂那。


.. code-block:: bash

   Using host libthread_db library "/lib/aarch64-linux-gnu/libthread_db.so.1".
   *********** loop 0 ************
   [New Thread 0x7fa7430db0 (LWP 9961)]
   [New Thread 0x7fa6c30db0 (LWP 9962)]
   [New Thread 0x7fa6430db0 (LWP 9963)]
   [New Thread 0x7fa5c30db0 (LWP 9964)]
   [New Thread 0x7fa5430db0 (LWP 9965)]
   [New Thread 0x7fa4c30db0 (LWP 9966)]
   [New Thread 0x7f8fffedb0 (LWP 9967)]
   [New Thread 0x7f8f7fedb0 (LWP 9968)]
   add total 12556 measurements.
   *********** loop 1 ************
   edges in graph: 12556
   iteration= 0	 chi2= 72633591.401419	 time= 0.735174	 cumTime= 0.735174	 edges= 12556	 schur= 0	 lambda= 12900954.939934	 levenbergIter= 1
   iteration= 1	 chi2= 62926900.131419	 time= 0.739193	 cumTime= 1.47437	 edges= 12556	 schur= 0	 lambda= 4300318.313311	 levenbergIter= 1
   iteration= 2	 chi2= 52845484.671121	 time= 0.736138	 cumTime= 2.2105	 edges= 12556	 schur= 0	 lambda= 1433439.437770	 levenbergIter= 1
   iteration= 3	 chi2= 45449292.558261	 time= 0.733896	 cumTime= 2.9444	 edges= 12556	 schur= 0	 lambda= 477813.145923	 levenbergIter= 1
   iteration= 4	 chi2= 38422805.114205	 time= 0.744241	 cumTime= 3.68864	 edges= 12556	 schur= 0	 lambda= 159271.048641	 levenbergIter= 1
   iteration= 5	 chi2= 31970398.890953	 time= 0.74077	 cumTime= 4.42941	 edges= 12556	 schur= 0	 lambda= 53090.349547	 levenbergIter= 1
   iteration= 6	 chi2= 24270565.530351	 time= 0.732403	 cumTime= 5.16182	 edges= 12556	 schur= 0	 lambda= 17696.783182	 levenbergIter= 1
   iteration= 7	 chi2= 12153446.174612	 time= 0.73939	 cumTime= 5.9012	 edges= 12556	 schur= 0	 lambda= 5898.927727	 levenbergIter= 1
   iteration= 8	 chi2= 5615434.148147	 time= 0.74188	 cumTime= 6.64308	 edges= 12556	 schur= 0	 lambda= 1966.309242	 levenbergIter= 1
   iteration= 9	 chi2= 4849512.586059	 time= 0.733603	 cumTime= 7.37669	 edges= 12556	 schur= 0	 lambda= 655.436414	 levenbergIter= 1
   iteration= 10	 chi2= 4785381.917957	 time= 0.733941	 cumTime= 8.11063	 edges= 12556	 schur= 0	 lambda= 218.478805	 levenbergIter= 1
   iteration= 11	 chi2= 4785319.436062	 time= 0.740044	 cumTime= 8.85067	 edges= 12556	 schur= 0	 lambda= 145.652536	 levenbergIter= 1
   iteration= 12	 chi2= 4785319.043803	 time= 1.63759	 cumTime= 10.4883	 edges= 12556	 schur= 0	 lambda= 1708231013067781.250000	 levenbergIter= 10

   The program being debugged has been started already.
   Start it from the beginning? (y or n) y
   Starting program: /srv/slambook/ch8/directMethod/build/direct_semidense data
   [Thread debugging using libthread_db enabled]
   Using host libthread_db library "/lib/aarch64-linux-gnu/libthread_db.so.1".
   *********** loop 0 ************
   [New Thread 0x7fa7430db0 (LWP 5767)]
   [New Thread 0x7fa6c30db0 (LWP 5768)]
   [New Thread 0x7fa6430db0 (LWP 5769)]
   [New Thread 0x7fa5c30db0 (LWP 5770)]
   [New Thread 0x7fa5430db0 (LWP 5771)]
   [New Thread 0x7fa4c30db0 (LWP 5772)]
   [New Thread 0x7f8fffedb0 (LWP 5773)]
   [New Thread 0x7f8f7fedb0 (LWP 5774)]
   add total 12556 measurements.
   *********** loop 1 ************
   edges in graph: 12556
   
   Thread 1 "direct_semidens" hit Breakpoint 2, poseEstimationDirect (measurements=std::vector of length 12556, capacity 16384 = {...}, gray=0x7fffffeef8, K=..., Tcw=...)
       at /srv/slambook/ch8/directMethod/direct_semidense.cpp:294
   294	    optimizer.optimize ( 30 );
   (gdb) b g2o::SparseOptimizer::optimize
   /build/gdb-qLNsm9/gdb-7.11.1/gdb/aarch64-tdep.c:334: internal-error: aarch64_analyze_prologue: Assertion `inst.operands[0].type == AARCH64_OPND_Rt' failed.
   A problem internal to GDB has been detected,
   further debugging may prove unreliable.
   Quit this debugging session? (y or n) 
   Please answer y or n.
   /build/gdb-qLNsm9/gdb-7.11.1/gdb/

   

进一步调试发现gdb也有问题，这时候基本挂在 return result上。 
   
.. code-block:: c
   
   operator Isometry3D() const                                                                                                                                                             ¦
      ¦288           {                                                                                                                                                                                       ¦
      ¦289             Isometry3D result = (Isometry3D) rotation();                                                                                                                                          ¦
      ¦290             result.translation() = translation();                                                                                                                                                 ¦
     >¦291             return result;                                           k::now();                                                                                                                    ¦
      ¦292           } 
   0x537838 <g2o::SE3Quat::operator Eigen::Transform<double, 3, 1, 0>() const+92>  str    x0, [sp]
   x0             0x7fffffdf18     549755805464
   x29            0x7fffffdf40     549755805504
   sp             0x7fffffdef0     0x7fffffdef0
     


同时进一步发现g2o 的优化时 **lamda** 异常提前退出

.. code-block:: bash

   [New Thread 0x7f8fffedb0 (LWP 5718)]
   [New Thread 0x7f8f7fedb0 (LWP 5719)]
   add total 12556 measurements.
   *********** loop 1 ************
   edges in graph: 12556
   
   Thread 1 "direct_semidens" hit Breakpoint 2, poseEstimationDirect (measurements=std::vector of length 12556, capacity 16384 = {...}, gray=0x7fffffeef8, K=..., Tcw=...)
       at /srv/slambook/ch8/directMethod/direct_semidense.cpp:294
   294	    optimizer.optimize ( 30 );
   (gdb) s optimizer.optimize(10)
   Couldn't find method g2o::SparseOptimizer::optimize
   
   /usr/local/include/g2o/core/base_unary_edge.h, /usr/local/include/g2o/core/base_edge.h, /usr/include/c++/5/bits/stl_set.h, /usr/include/c++/5/bits/ptr_traits.h, 
   /usr/local/include/g2o/core/sparse_block_matrix_diagonal.h, /usr/local/include/g2o/core/block_solver.h, /usr/include/c++/5/bits/unordered_map.h, /usr/include/c++/5/bits/hashtable.h, 
   /usr/include/c++/5/utility, /usr/include/c++/5/bits/functional_hash.h, /usr/include/c++/5/bits/hashtable_policy.h, /usr/local/include/g2o/core/sparse_block_matrix_ccs.h, 
   /usr/include/c++/5/bits/stl_map.h, /usr/include/c++/5/bits/stl_function.h, /usr/include/c++/5/ext/aligned_buffer.h, /usr/local/include/g2o/core/sparse_block_matrix.h, 
   /usr/local/include/g2o/core/linear_solver.h, /usr/include/c++/5/chrono, /usr/include/c++/5/bits/stl_iterator_base_types.h, /usr/include/c++/5/bits/stl_iterator.h, /usr/include/c++/5/bits/stl_algo.h, 
   ---Type <return> to continue, or q <return> to quit---q
   /usrQuit
   (gdb) b g2o::SparseOptimizer::optimize
   /build/gdb-qLNsm9/gdb-7.11.1/gdb/aarch64-tdep.c:334: internal-error: aarch64_analyze_prologue: Assertion `inst.operands[0].type == AARCH64_OPND_Rt' failed.
   A problem internal to GDB has been detected,
   further debugging may prove unreliable.
   Quit this debugging session? (y or n) y


   //gdb source code 
   /* Analyze a prologue, looking for a recognizable stack frame
      and frame pointer.  Scan until we encounter a store that could
      clobber the stack frame unexpectedly, or an unknown instruction.  */
   
   static CORE_ADDR
   aarch64_analyze_prologue (struct gdbarch *gdbarch,
   			  CORE_ADDR start, CORE_ADDR limit,
   			  struct aarch64_prologue_cache *cache)
   {
     enum bfd_endian byte_order_for_code = gdbarch_byte_order_for_code (gdbarch);
     int i;
     pv_t regs[AARCH64_X_REGISTER_COUNT];
     struct pv_area *stack;
     struct cleanup *back_to;
   
     for (i = 0; i < AARCH64_X_REGISTER_COUNT; i++)
       regs[i] = pv_register (i, 0);
     stack = make_pv_area (AARCH64_SP_REGNUM, gdbarch_addr_bit (gdbarch));
     back_to = make_cleanup_free_pv_area (stack);
   
     for (; start < limit; start += 4)
       {
         uint32_t insn;
         aarch64_inst inst;
   
         insn = read_memory_unsigned_integer (start, 4, byte_order_for_code);
   
         if (aarch64_decode_insn (insn, &inst, 1) != 0)
   	break;
   
         if (inst.opcode->iclass == addsub_imm
   	  && (inst.opcode->op == OP_ADD
   	      || strcmp ("sub", inst.opcode->name) == 0))
   	{
   	  unsigned rd = inst.operands[0].reg.regno;
   	  unsigned rn = inst.operands[1].reg.regno;
   
   	  gdb_assert (aarch64_num_of_operands (inst.opcode) == 3);
   	  gdb_assert (inst.operands[0].type == AARCH64_OPND_Rd_SP);
   	  gdb_assert (inst.operands[1].type == AARCH64_OPND_Rn_SP);
   	  gdb_assert (inst.operands[2].type == AARCH64_OPND_AIMM);
   
   	  if (inst.opcode->op == OP_ADD)
   	    {
   	      regs[rd] = pv_add_constant (regs[rn],
   					  inst.operands[2].imm.value);
   	    }
   	  else
   	    {
   	      regs[rd] = pv_add_constant (regs[rn],
   					  -inst.operands[2].imm.value);
   	    }
   	}
         else if (inst.opcode->iclass == pcreladdr
   	       && inst.operands[1].type == AARCH64_OPND_ADDR_ADRP)
   	{
   	  gdb_assert (aarch64_num_of_operands (inst.opcode) == 2);
   	  gdb_assert (inst.operands[0].type == AARCH64_OPND_Rd);
   
   	  regs[inst.operands[0].reg.regno] = pv_unknown ();
   	}
         else if (inst.opcode->iclass == branch_imm)
   	{
   	  /* Stop analysis on branch.  */
   	  break;
   	}
         else if (inst.opcode->iclass == condbranch)
   	{
   	  /* Stop analysis on branch.  */
   	  break;
   	}
         else if (inst.opcode->iclass == branch_reg)
   	{
   	  /* Stop analysis on branch.  */
   	  break;
   	}
         else if (inst.opcode->iclass == compbranch)
   	{
   	  /* Stop analysis on branch.  */
   	  break;
   	}
         else if (inst.opcode->op == OP_MOVZ)
   	{
   	  gdb_assert (inst.operands[0].type == AARCH64_OPND_Rd);
   	  regs[inst.operands[0].reg.regno] = pv_unknown ();
   	}
         else if (inst.opcode->iclass == log_shift
   	       && strcmp (inst.opcode->name, "orr") == 0)
   	{
   	  unsigned rd = inst.operands[0].reg.regno;
   	  unsigned rn = inst.operands[1].reg.regno;
   	  unsigned rm = inst.operands[2].reg.regno;
   
   	  gdb_assert (inst.operands[0].type == AARCH64_OPND_Rd);
   	  gdb_assert (inst.operands[1].type == AARCH64_OPND_Rn);
   	  gdb_assert (inst.operands[2].type == AARCH64_OPND_Rm_SFT);
   
   	  if (inst.operands[2].shifter.amount == 0
   	      && rn == AARCH64_SP_REGNUM)
   	    regs[rd] = regs[rm];
   	  else
   	    {
   	      if (aarch64_debug)
   		{
   		  debug_printf ("aarch64: prologue analysis gave up "
   				"addr=0x%s opcode=0x%x (orr x register)\n",
   				core_addr_to_string_nz (start), insn);
   		}
   	      break;
   	    }
   	}
         else if (inst.opcode->op == OP_STUR)
   	{
   	  unsigned rt = inst.operands[0].reg.regno;
   	  unsigned rn = inst.operands[1].addr.base_regno;
   	  int is64
   	    = (aarch64_get_qualifier_esize (inst.operands[0].qualifier) == 8);
   
   	  gdb_assert (aarch64_num_of_operands (inst.opcode) == 2);
   	  gdb_assert (inst.operands[0].type == AARCH64_OPND_Rt);
   	  gdb_assert (inst.operands[1].type == AARCH64_OPND_ADDR_SIMM9);
   	  gdb_assert (!inst.operands[1].addr.offset.is_reg);
   
   	  pv_area_store (stack, pv_add_constant (regs[rn],
   						 inst.operands[1].addr.offset.imm),
   			 is64 ? 8 : 4, regs[rt]);
   	}
         else if ((inst.opcode->iclass == ldstpair_off
   		|| inst.opcode->iclass == ldstpair_indexed)
   	       && inst.operands[2].addr.preind
   	       && strcmp ("stp", inst.opcode->name) == 0)
   	{
   	  unsigned rt1 = inst.operands[0].reg.regno;
   	  unsigned rt2 = inst.operands[1].reg.regno;
   	  unsigned rn = inst.operands[2].addr.base_regno;
   	  int32_t imm = inst.operands[2].addr.offset.imm;
   
   	  gdb_assert (inst.operands[0].type == AARCH64_OPND_Rt);
   	  gdb_assert (inst.operands[1].type == AARCH64_OPND_Rt2);
   	  gdb_assert (inst.operands[2].type == AARCH64_OPND_ADDR_SIMM7);
   	  gdb_assert (!inst.operands[2].addr.offset.is_reg);
   
   	  /* If recording this store would invalidate the store area
   	     (perhaps because rn is not known) then we should abandon
   	     further prologue analysis.  */
   	  if (pv_area_store_would_trash (stack,
   					 pv_add_constant (regs[rn], imm)))
   	    break;
   
   	  if (pv_area_store_would_trash (stack,
   					 pv_add_constant (regs[rn], imm + 8)))
   	    break;
   
   	  pv_area_store (stack, pv_add_constant (regs[rn], imm), 8,
   			 regs[rt1]);
   	  pv_area_store (stack, pv_add_constant (regs[rn], imm + 8), 8,
   			 regs[rt2]);
   
   	  if (inst.operands[2].addr.writeback)
   	    regs[rn] = pv_add_constant (regs[rn], imm);
   
   	}
         else if (inst.opcode->iclass == testbranch)
   	{
   	  /* Stop analysis on branch.  */
   	  break;
   	}
         else
   	{
   	  if (aarch64_debug)
   	    {
   	      debug_printf ("aarch64: prologue analysis gave up addr=0x%s"
   			    " opcode=0x%x\n",
   			    core_addr_to_string_nz (start), insn);
   	    }
   	  break;
   	}
       }
   
     if (cache == NULL)
       {
         do_cleanups (back_to);
         return start;
       }
   
     if (pv_is_register (regs[AARCH64_FP_REGNUM], AARCH64_SP_REGNUM))
       {
         /* Frame pointer is fp.  Frame size is constant.  */
         cache->framereg = AARCH64_FP_REGNUM;
         cache->framesize = -regs[AARCH64_FP_REGNUM].k;
       }
     else if (pv_is_register (regs[AARCH64_SP_REGNUM], AARCH64_SP_REGNUM))
       {
         /* Try the stack pointer.  */
         cache->framesize = -regs[AARCH64_SP_REGNUM].k;
         cache->framereg = AARCH64_SP_REGNUM;
       }
     else
       {
         /* We're just out of luck.  We don't know where the frame is.  */
         cache->framereg = -1;
         cache->framesize = 0;
       }
   
     for (i = 0; i < AARCH64_X_REGISTER_COUNT; i++)
       {
         CORE_ADDR offset;
   
         if (pv_area_find_reg (stack, gdbarch, i, &offset))
   	cache->saved_regs[i].addr = offset;
       }
   
     do_cleanups (back_to);
     return start;
   }



实质上采用的是 levenbergIter迭代法,遇到这种问题，就要看返回的对象是不是对，是不是因为未定状态，造成stackoverflow，从而引发的 trap的signal. 例如返回的result的值是否合法。

因为 aarch64 中 x29 就是Framepointer. http://infocenter.arm.com/help/topic/com.arm.doc.ihi0055b/IHI0055B_aapcs64.pdf

.. code-block:: asm

   0x537838 <g2o::SE3Quat::operator Eigen::Transform<double, 3, 1, 0>() const+92>  str    x0, [sp]
   0x537838 <g2o::SE3Quat::operator Eigen::Transform<double, 3, 1, 0>() const+92>  mov    sp, x29
   x0             0x7fffffdf18     549755805464


.. code-block:: c

   void OptimizationAlgorithmLevenberg::printVerbose(std::ostream& os) const
    {
      os
        << "\t schur= " << _solver->schur()
        << "\t lambda= " << FIXED(_currentLambda)
        << "\t levenbergIter= " << _levenbergIterations;
    }
   
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

build error: gtsam module 为什么版本之间ARM不兼容。 https://github.com/introlab/rtabmap/issues/131，
即使都采用了相同的编译器与C++11 还是有这样的问题，很有可能是一些宏导致构成cast 链出现问题。

pose_graph_g2o_lie
------------------

.. figure:: /Stage_3/SLAM_book/ch11_pose_graph_g2o_lie.png
   
   unkown的callstack就需要进一步用的工具分析。或者添加相关的库的dbg文件来进一步解析。

.. figure:: /Stage_3/SLAM_book/ch11_pose_graph_g2o_lie_timeline.png
   cholmod_factorize 是blas库的 CHolesky分解也就是LU分解。

.. figure:: /Stage_3/SLAM_book/ch11_pose_graph_g2o_lie_timeline_topdown.png


ch12
====

字典的生成，DBoW3。 字典建立的原理，

#. 从训练图像中离线抽取特征
#. 将抽取的特征用聚类算法把描述子空间划分成K类。
#. 将划分的每个子空间，继续利用聚类分析
#. 直到将描述子形成一个树形结构


.. code-block:: c

   struct Node {
      NodeId id;  //节点标号
      WordValue  weight; // 该节点的权重，也出现的字数除以总描述子数目
      TDescriptor descriptor; //描述符
      WordId word_id;  //如果是叶子节点，则有词汇的id
   }

DBow#的结构，

`Fast Bag of Words <https://github.com/rmsalinas/fbow>`_ 用AVX,SSE,MMX 指令来优化了一下，加载 比DBOW2快了~80倍，
生成字典时快了 ~6.4倍。


feature_training
----------------

.. image:: /Stage_3/SLAM_book/ch12_feature_training_Analysis_summary.png


.. figure:: /Stage_3/SLAM_book/ch12_feature_training_timeline.png

   虽然有多线程，但是并行度非常低，并且thread context的切换次数比较多。

Loop_check
----------

.. image:: /Stage_3/SLAM_book/ch12_loop_check_Analysis_summary.png

.. image:: /Stage_3/SLAM_book/ch12_loop_check_timeline.png

ch13
====

octomap 是可以用来展示地图，https://github.com/OctoMap/octomap
几种地图的优缺点 http://www.cnblogs.com/gaoxiang12/p/5041142.html

而简单的pcd 点运地图没有法进行导房，而采用了八叉树的模式来建立地图就像游戏地图的构件过程一样。


dense_monocular
----------------

直接计算特征点匹配计算量太大，采用极线搜索与块比匹配。
