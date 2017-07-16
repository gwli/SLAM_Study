
基本的工作流
============

.. graphviz:: 
   
   digraph G {
	graph [layout=dot rankdir=LR];
        node [shape=box];
        sensor->vo->nlp->map[weight=10];
        sensor->loop_check->map;
   }
