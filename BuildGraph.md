# BuildGraph

## 1. 什么是BuildGraph

    1-1.BG是一个用xml编写的，makefile-like的自动构建系统。其功能图由构建块组成。

## 2. 组成元素

    2-1. Task：组成构建过程的执行动作
    2-2. Node：一个有名 task 序列。
    2-3. Agents：一组执行在相同机器上的节点。对本地构建无效果。
    2-4. Trigger：需要人工干预时，会执行的容器。
    2-5. Aggregate：能被一个名称应用的一组 nodes 和有名输出。