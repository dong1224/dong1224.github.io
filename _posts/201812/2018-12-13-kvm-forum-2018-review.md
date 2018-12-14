# kvm forum 2018概述

给自己挖个坑，总结下kvm forum 2018的最新成果。

## 1. Guest Free Page Hinting - Nitesh Narayan Lal, Red Hat

当主机内存不足时，释放客户机无用内存。
当前情况是虚拟机释放的内存不会通知到host主机。现在里面是有个ballon机制在。

还在做，目前对每次释放内存都要这么算一下，会增加cpu。优化方向是加个算法上去，到一定量的内存再通知主机回收。

## 2. 