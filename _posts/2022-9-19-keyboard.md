---
layout: post
title: verilog键盘输入示例代码及分析（摩尔型有限状态机）
subtitle: “一生一芯”项目笔记
author: 永清
categories: verilog
banner:
  image: /assets/images/banners/4.jpg
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: verilog
---

> 往昔鸳鸯戏水，而今不相依偎，美景良辰纵然抚媚亦徒留伤悲。----《美人画卷》

---

本代码是一生一芯项目中，南京大学nvboard开源项目键盘扫描示例代码。我们抛开上层连接不谈，分析一下这个代码。同时我自己也理清一下思路，不然总是感觉些许混乱，或者说，明明用51单片机写的时序接收这么好理解，为什么这个程序我没有一眼看出来他在干什么。因为这段代码本身确实蕴含着一个设计思想，请务必读到最后。<br>
南京大学数电实验网站：[点此](https://nju-projectn.github.io/dlco-lecture-note/index.html)<br>
nvboard GitHub（即本文源码出处）：https://github.com/NJU-ProjectN/nvboard.git

**源代码著作权归原作者所有。本文完全尊重原作者的著作权，仅引用、注释以供学习，不参与网站创作激励，不具有商业、盈利性质。**

## 0. 源代码

```cpp
module ps2_keyboard(clk,resetn,ps2_clk,ps2_data);
    input clk,resetn,ps2_clk,ps2_data;

    reg [9:0] buffer;        // ps2_data bits
    reg [3:0] count;  // count ps2_data bits
    reg [2:0] ps2_clk_sync;

    always @(posedge clk) begin
        ps2_clk_sync <=  {ps2_clk_sync[1:0],ps2_clk};
    end

    wire sampling = ps2_clk_sync[2] & ~ps2_clk_sync[1];

    always @(posedge clk) begin
        if (resetn == 0) begin // reset
            count <= 0;
        end
        else begin
            if (sampling) begin
              if (count == 4'd10) begin
                if ((buffer[0] == 0) &&  // start bit
                    (ps2_data)       &&  // stop bit
                    (^buffer[9:1])) begin      // odd  parity
                    $display("receive %x", buffer[8:1]);
                end
                count <= 0;     // for next
              end else begin
                buffer[count] <= ps2_data;  // store ps2_data
                count <= count + 3'b1;
              end
            end
        end
    end

endmodule
```

## 1. 相关知识

键盘传输过来的有两个数据，分别是`ps2_clk,ps2_data`，即键盘的时钟，以及按键的通码断码。<br>
**扫描码**：被规定的、某键被按下/弹起时所送出的代码。<br>
**通码**是：约定的、按键按下是键盘送出的编码，现代键盘基本统一。<br>
**断码**是：`F0h`加此按键的通码，如释放 W 键时输出的断码为`F0h 1Dh`，分两帧传输。<br>

下图是键盘扫描表，原图出自南京大学数字电子电路实验课程网站，更多资料可以上网去搜。
![键盘扫描码](/assets/images/6/1.png)

下图是传输时序图，原图出自南京大学数字电子电路实验课程网站，[网址点此](https://nju-projectn.github.io/dlco-lecture-note/exp/07.html)
![传输](/assets/images/6/2.png)
即：一帧数据共9位，最后一位P是奇偶校验（奇校验）。

## 2.代码分析
代码先定义了一个三位的时钟变量`ps2_clk_sync`，通过一个死循环，每次把`ps2_clk`时钟上的采样送入最低位，并丢弃最高位，即形成了一个三位的时间队列。
因为每次最新的采样放入的是最低位0位，所以计算的`ps2_clk_sync[2] & ~ps2_clk_sync[1]`。若为下降沿，则sampling为1，上升沿为0。

但是读到这里相信都不免有些疑问：
为什么不`ps2_clk_sync[1] & ~ps2_clk_sync[0]`？这样岂不更加及时？或者直接把`ps2_clk`设置成always的敏感变量，不是更一步到位？这样做的意义是什么？
请先思考，文末有笔者认为的原因。
```cpp
reg [2:0] ps2_clk_sync;

always @(posedge clk) begin
        ps2_clk_sync <=  {ps2_clk_sync[1:0],ps2_clk};
    end
    
wire sampling = ps2_clk_sync[2] & ~ps2_clk_sync[1];
```
---
如此，则sampling内存放的是当前是否是下降沿；若当前是下降沿，则判断了`count == 4'd10`，count就是个计数器，每次`count <= count + 3'b1;`，所以就是判断是否接收到了十位。看清这里是`4'd10`就好，我之前没看清以为是`4'd10`卡了半天。然后就是判断首位是否为0、1，并且使用了校验。
这里可以学习一下他写的奇偶校验，`^buffer[9:1]`，需要注意的是这里的`^`是**缩减运算符**，其运算方式是
```dark
ans = ((buffer[1] ^ buffer[2]) ^ buffer[3])^ ······
```
即从最低位开始向上计算。最后的结果也是一个布尔值。
如果上述判断条件都通过，就输出buffer。

但是你是否有这样的疑问：第十位是stop空位，并且奇偶校验在第九次收到之后就可以进行了，为什么非要等这个空的帧入队之后，第十一次才开始判断呢？
```cpp
 reg [9:0] buffer;        // ps2_data bits
 reg [3:0] count;  // count ps2_data bits

 always @(posedge clk) begin
        if (resetn == 0) begin // reset
            count <= 0;
        end
        else begin
            if (sampling) begin
              if (count == 4'd10) begin
                if ((buffer[0] == 0) &&  // start bit
                    (ps2_data)       &&  // stop bit
                    (^buffer[9:1])) begin      // odd  parity
                    $display("receive %x", buffer[8:1]);
                end
                count <= 0;     // for next
              end else begin
                buffer[count] <= ps2_data;  // store ps2_data
                count <= count + 3'b1;
              end
            end
        end
    end

```

## 3. 摩尔型有限状态机
南大这个网站把状态机和键盘放在一起，让我很是不理解，明明两个风马牛不相及的东西，放一起干什么，但是仔细一想，其实是因为这段代码使用了摩尔型有限状态机的思想。
下面摘录一下摩尔型状态机的知识，同样参考南大的数电实验网站。
> Moore 型有限状态机的输出信号只与有限状态机的当前状态有关，与输入信号的当前值无关，输入信号的当前值只会影响到状态机的次态，不会影响状态机当前的输出。即Moore 型有限状态机的输出信号是直接由状态寄存器译码得到。 Moore型有限状态机在时钟CLK信号有效后经过一段时间的延迟，输出达到稳定值。即使在这个时钟周期内输入信号发生变化，输出也会在这个完整的时钟周期内保持稳定值而不变。输入对输出的影响要到下一个时钟周期才能反映出来。Moore有限状态机最重要的特点就是将输入与输出信号隔离开来。
> 
观察这段代码，不难发现使用的就是Moore（摩尔）型有限状态机的思想。因为：
1. 它对时间序列进行了一个存储，当前时序判断前两个时序是否存在下降沿，而不是接收到时钟信号立即判断下降沿。
2. 对于buffer也不是收到第十位(count == 9)就立即输出，而是第十位也放入buffer内后才进行校验、输出。如果存在第十一位的话，其实现在已经是第十一位了。

看来前面我们对为什么要存储时间序列的疑问解决了。甚好。

> 回忆斑驳微醉，叹相思未随，几春几秋几段轮回。----《美人画卷》