---
layout: post
title: 卡尔曼滤波学习笔记3（没写完）
subtitle: 一维无过程噪声的卡尔曼滤波,以及多维卡尔曼滤波的基础知识
categories: math
banner:
  image: /assets/images/banners/11.jpg
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: KarmanFilter math
---

## 状态外插方程

&emsp;&emsp;使用状态外插方程，能够基于当前系统状态预测下一个系统状态。它把当前n时刻的状态向量外插至未来n+1时刻。状态外插方程的一般形式是：

$$\boldsymbol{\hat{x}}_{n+1,n} = \boldsymbol{F\hat{x}}_{n,n} + \boldsymbol{Gu}_{n} + \boldsymbol{w}_{n}$$

其中：

$\boldsymbol{\hat{x}}_{n+1,n}$&emsp;&emsp;是n时刻对n+1时刻状态的预测

$\boldsymbol{\hat{x}}_{n,n}$&emsp;&emsp;是n时刻系统状态向量的估计

$u_n$&emsp;&emsp;是 控制向量 或 输入向量 - 该系统的一个 可测量的（确定性的）输入

$w_n$&emsp;&emsp;是 过程噪声 或 扰动 - 能够影响系统状态的 不可测量的 输入

$\boldsymbol{F}$&emsp;&emsp;是 状态转移矩阵

$\boldsymbol{G}$&emsp;&emsp;是 控制矩阵 或 输入转移矩阵 （将控制量映射到状态变量上）

### 例1. 匀加速运动的飞机

&emsp;&emsp;考虑一个在三位空间匀加速运动的飞机，其状态向量为：

$$\boldsymbol{\hat{x}}_{n}=
\left[ \begin{matrix}
        \hat{x}_{n}\\
        \hat{y}_{n}\\
        \hat{z}_{n}\\
        \hat{\dot{x}}_{n}\\
        \hat{\dot{y}}_{n}\\
        \hat{\dot{z}}_{n}\\
        \hat{\ddot{x}}_{n}\\
        \hat{\ddot{y}}_{n}\\
        \hat{\ddot{z}}_{n}\\
\end{matrix}
\right]$$

&emsp;&emsp;状态转移矩阵是：

$$\boldsymbol{F}=  
\left[ \begin{matrix}
1 & 0 & 0 & \Delta t & 0 		& 0			& 0.5\Delta t^{2} 	& 0 				& 0 \\
0 & 1 & 0 & 0 		 & \Delta t & 0			& 0 				& 0.5\Delta t^{2} 	& 0 \\
0 & 0 & 1 & 0 		 & 0 		& \Delta t	& 0 				& 0 				& 0.5\Delta t^{2}\\
0 & 0 & 0 & 1 		 & 0 		& 0			& \Delta t 			& 0 				& 0 \\
0 & 0 & 0 & 0 		 & 1		& 0			& 0 				& \Delta t			& 0 \\
0 & 0 & 0 & 0 		 & 0 		& 1			& 0 				& 0 				& \Delta t\\
0 & 0 & 0 & 0 		 & 0 		& 0			& 1 				& 0 				& 0 \\
0 & 0 & 0 & 0 		 & 0		& 0			& 0 				& 1					& 0 \\
0 & 0 & 0 & 0 		 & 0 		& 0			& 0 				& 0 				& 1\\
\end{matrix}
\right]$$

### 例2. 读取加速度的飞机

&emsp;&emsp;此方程由基本运动学规律得到。考虑这样一个情况：在其它情况和上一个例子相同的情况下，飞机座舱内有一个操纵杆，我们直接知道驾驶员的加速指令。那么，加速度显然不能再写入待估计的状态向量中了。此时，状态向量是：

$$\boldsymbol{\hat{x}}_{n}=
\left[ \begin{matrix}
\hat{x}_{n}\\
\hat{y}_{n}\\
\hat{z}_{n}\\
\hat{\dot{x}}_{n}\\
\hat{\dot{y}}_{n}\\
\hat{\dot{z}}_{n}\\
\end{matrix}
\right]$$

&emsp;&emsp;描述测量到的飞机加速度向量（控制向量）是：

$$\boldsymbol{u}_{n}=  
\left[ \begin{matrix}
\ddot{x}_{n}\\
\ddot{y}_{n}\\
\ddot{z}_{n}\\
\end{matrix}
\right]$$

&emsp;&emsp;状态转移矩阵是：

$$\boldsymbol{F}=  
\left[ \begin{matrix}
1 & 0 & 0 & \Delta t & 0 		& 0\\
0 & 1 & 0 & 0 		 & \Delta t & 0\\
0 & 0 & 1 & 0 		 & 0 		& \Delta t\\
0 & 0 & 0 & 1 		 & 0 		& 0\\
0 & 0 & 0 & 0 		 & 1		& 0\\
0 & 0 & 0 & 0 		 & 0 		& 1\\
\end{matrix}
\right]$$

&emsp;&emsp;控制矩阵是：

$$\boldsymbol{G}=  
\left[ \begin{matrix}
0.5\Delta t^{2} & 0 				& 0 \\
0 				& 0.5\Delta t^{2}	& 0 \\
0 				& 0 				& 0.5\Delta t^{2} \\
\Delta t 		& 0 				& 0 \\
0 				& \Delta t 			& 0 \\
0 				& 0 				& \Delta t \\
\end{matrix}
\right]$$

&emsp;&emsp;此时的状态外插方程为：

$$\boldsymbol{\hat{x}}_{n+1,n} = \boldsymbol{F\hat{x}}_{n,n} + \boldsymbol{Gu}_{n,n}$$

&emsp;&emsp;注意为什么控制矩阵的下面还有三个 $\Delta t$ ，因为加速度除了对位置xyz产生影响，也对速度产生影响，控制矩阵的下面三行对应着速度状态。

### 练习1. 自由落体运动

&emsp;&emsp;不妨对自由落体运动进行建模考察。首先，其状态向量为：

$$\boldsymbol{\hat{x}}_{n}=
\left[ \begin{matrix}
\hat{h}_{n}\\
\hat{\dot{h}}_{n}\\
\end{matrix}
\right]$$

&emsp;&emsp;状态转移矩阵是：

$$\boldsymbol{F}=  
\left[ \begin{matrix}
1 & \Delta t \\
0 & 1 \\
\end{matrix}
\right]$$

&emsp;&emsp;控制向量是：

$$\boldsymbol{u}_{n}=  
								\left[ \begin{matrix}								
									g
								\end{matrix}
								 \right]$$

&emsp;&emsp;控制矩阵是：

$$\boldsymbol{G}=  
								\left[ \begin{matrix}								
									0.5\Delta t^{2} \\
									\Delta t \\
								\end{matrix}
								 \right]$$

&emsp;&emsp;我发现，在作控制矩阵的时候，是仅仅考虑控制向量**相对于**上一个时刻的影响，什么意思，就是把上一个状态看作是零状态。比如说吧，匀加速直线运动，在作状态矩阵的时候，加速度对位移的影响就是二分之一at平方，这是把看作是从上一个时刻开始的初速度为零的匀加速运动了。我还以为要作差求解。这里一定要辨明。

