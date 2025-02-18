---
title: 在FPGA上实现多读多写存储器
weight: 2
---

## 背景

计算和带宽是一对亟需平衡的矛盾。高并行度的计算如果没有高并行度的访存，就大概率会遇到速度瓶颈。在Xilinx FPGA上，Xilinx提供的BRAM IP最高只能实现真双端口RAM。为了在FPGA上实现更多端口的同时读写，尝试了以下几种方法。

## FPGA的片上存储资源

要设计存储器，首先需要了解FPGA有哪些存储资源可利用。主要包括两部分：CLB Slice中的寄存器和查找表、Block RAM。Block RAM是专门用于存储的资源，量大价优。而寄存器则数量有限，生成的RAM被称为Distributed RAM。我们在Verilog中编写的reg如果综合成了触发器，用的便是CLB Slice中的寄存器资源。

## 方法1：用寄存器组作为存储器，实现任意读写

如果直接用reg变量生成存储器，我们实际上可以实现任意并行度的读写。然而这种方式的缺点是极其占用FPGA的寄存器资源，挤压了其他逻辑的空间。因此无法满足较大的存储需求。

## 方法2：用Distributed RAM作为存储器，实现多读二写

使用reg变量，按照RAM的方式来写存储器，并加入(* ram_style = “distributed” *)引导综合工具综合成Distributed RAM。这一方法能够实现任意读（似乎是？我测试了16读的情况），但在综合报告中显示最多只能支持二端口写，仍有一定局限性。并且使用的仍然是CLB Slice中的寄存器资源，虽然相比方法1消耗得少一些，但无法满足太大的存储需求。

## 方法3：用Block RAM作为存储器，实现多读多写

最终我采用的是方法3。Xilinx官方的BRAM IP仅支持二端口读二端口写。一种比较直观方法是将N个单端口BRAM联合起来，组成多端口的存储器。如下图所示：

![image-20220530151835263](https://tuchuang-1254351169.cos.ap-guangzhou.myqcloud.com/image-20220530151835263.png)

图中为二端口读二端口写的方案，可以拓展为更多端口的存储器。

然而这种方法的问题在于可能出现Bank Conflict冲突。也就是多个读/写端口可能同时访问同一个BRAM，这时BRAM的单端口就无法满足读写请求。这种情况需要加入仲裁机制，对于冲突情况退回一部分请求，直到前一个读写完成。实现要稍微复杂一点。同时，为了实现并行读写，采用了大量的组合逻辑连线。必须妥善处理前一次读写后的信号线，否则前一次读写的信号可能与下一次读写的信号发生竞争冒险，造成读写出错。如果有同时读写的情况，就需要引入更复杂的机制。

## 更多方法......

使用Block RAM搭建多端口读写的RAM已经有很多研究成果，也有相关开源项目。

具体可以参考：

1、https://tomverbeure.github.io/2019/08/03/Multiport-Memories.html

2、http://fpgacpu.ca/multiport/
