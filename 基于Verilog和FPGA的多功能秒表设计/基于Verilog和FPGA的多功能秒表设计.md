# 基于Verilog和FPGA的多功能秒表设计
## 实验目的

1. 初步掌握利用Verilog硬件描述语言进行逻辑功能设计的原理和方法。
2. 理解和掌握运用大规模可编程逻辑器件进行逻辑设计的原理和方法。
3. 理解硬件实现方法中的并行性，联系软件实现方法中的并发性。
4. 理解硬件和软件是相辅相成、并在设计和应用方法上的优势互补的特点。
5. 本实验学习积累的Verilog硬件描述语言和对FPGA/CPLD的编程操作，是进行后续《计算机组成原理》部分课程实验，设计实现计算机逻辑的基础。

## 实验内容

1. 运用Verilog硬件描述语言，基于DE1-SOC实验板，设计实现一个具有较多功能的计时秒表。
2. 要求将8个数码管设计为具有“时：分：秒：毫秒”显示，按键的基本控制动作有3个：“计时复位”、“计数/暂停”、“显示暂停/显示继续”。功能能够满足马拉松或长跑运动员的计时需要。
3. 利用示波器观察按键的抖动，设计按键电路的消抖方法。
4. 在实验报告中详细报告自己的设计过程、步骤及Verilog代码。

## 实验仪器

Altera – DE1-SOC实验板 1套
示波器 1台
数字万用表 1台

## 实验任务

#### 实验电路

![1558584325650](C:\Users\David Stark\AppData\Roaming\Typora\typora-user-images\1558584325650.png)

![1558584340614](C:\Users\David Stark\AppData\Roaming\Typora\typora-user-images\1558584340614.png)

#### 基本功能

以下是秒表的计时和暂停等功能：

- 最开始的代码部分加入了防抖动的功能。
- when循环过程中，0.5M表示10ms进行一次毫秒数值的计数。
- 按键在按下的时候，进行按键反转（表现在led灯上面就是灯的明暗变化）
- 循环此过程

```verilog
begin
    if(key_start_pause==0)
        begin
            counter_start <= counter_start+1;
            if(counter_start>=1000000 && counter_start<1500000)
                start_1_time <= 1;
            else 
                start_1_time <= 0;
        end
    else 
        counter_start<=0;

    if(key_reset==0)
        begin	
            counter_reset<=counter_reset+1;
            if(counter_reset>=1000000 && counter_reset< 1500000)
                reset_1_time<=1;
            else 
                reset_1_time<=0;
        end
    else 
        counter_reset<=0;

    if(key_display_stop==0)
        begin
            counter_display <=counter_display+1;
            if(counter_display>=1000000 && counter_display< 1500000)
                display_1_time<=1;
            else	
                display_1_time<=0;
        end
    else
        counter_display<=0;

    if (counter_50M == 500000)
        begin

            if(key_reset==0 && reset_1_time==1)
                begin
                    msecond_counter_low <=0;
                    msecond_counter_high <=0;
                    second_counter_low<=0;
                    second_counter_high<=0;
                    minute_counter_low<=0;	
                    minute_counter_high<=0;
                    led3<=0;
                end

            if(key_start_pause==0 && start_1_time == 1)
                begin
                    led1 <= !led1;
                end

            if(key_display_stop==0 && display_1_time == 1)
                begin
                    led2 <= !led2;
                end

            if(key_reset==1)
                begin
                    led0<=1;
                end

            if (led1==1) 
                begin
                    msecond_counter_low <= msecond_counter_low +1;
                    if(msecond_counter_low==9)
                        begin
                            led3<=!led3;
                            msecond_counter_low<=0;
                            msecond_counter_high<=msecond_counter_high+1;
                            if(msecond_counter_high==9)
                                begin
                                    msecond_counter_high<=0;
                                    second_counter_low<=second_counter_low+1;
                                    if(second_counter_low==9)
                                        begin 
                                            second_counter_low<=0;
                                            second_counter_high<=second_counter_high+1;
                                            if(second_counter_high==5)
                                                begin
                                                    second_counter_high<=0;
                                                    minute_counter_low<=minute_counter_low+1;
                                                    if(minute_counter_low==9)
                                                        begin
                                                            minute_counter_low<=0;
                                                            minute_counter_high<=minute_counter_high+1;
                                                            if (minute_counter_high==5)
                                                                begin
                                                                    minute_counter_high<=0;
                                                                end
                                                        end
                                                end
                                        end
                                end
                        end	
                end	

            if (led2==0)
                begin 
                    minute_display_high <= minute_counter_high;
                    minute_display_low <= minute_counter_low;
                    second_display_high <= second_counter_high;
                    second_display_low <= second_counter_low;
                    msecond_display_high <= msecond_counter_high;
                    msecond_display_low <= msecond_counter_low;
                end
            counter_50M <= 0;
        end
    else counter_50M <= counter_50M + 1;
end
```

#### 消抖

当系统检测出按键闭合后，执行一个延时程序，产生5ms～10ms的延时；前沿抖动消失后，再一次检测键的状态；如果仍保持闭合状态电平，则确认为真正有键按下。当检测到按键释放后，也要给5ms～10ms的延时，待后沿抖动消失后才能转入该键的处理程序。设置经过20 ms后的高电平才是真正的按键功能。

我的设计通过`start_1_time == 1`（以start/pause为例进行解释）来表明按键有效，通过`(key_start_pause==0 && start_1_time == 1)`来进行按键的判断。

同时延时设计为 1M-1.5M ，可以通过这种方式进行消抖，且唯一激活按键的操作。

```verilog
if(key_start_pause==0)
    begin
        counter_start <= counter_start+1;
        if(counter_start>=1000000 && counter_start<1500000)
            start_1_time <= 1;
        else 
            start_1_time <= 0;
    end
else 
    counter_start<=0;
```
## 实验总结
这次实验收获还是蛮大的，进行实验的操作是一个循序渐进的过程。我先完成对秒表功能的设计，然后加入防抖的功能。防抖也进行了几次尝试和探索，简述如下：

1. 第一次使用 `counter>=1M`进行判断：这个是是考虑到了延时，并且对按键进行操作激活。但是发现在之后的循环中按键由于大于1M的时间都是激活状态，导致奇数次操作有效，偶数次无效。所以有1/2的几率操作不被激活。故放弃此操作。
2. 第二次使用`counter==1M`进行判断：但是发现由于起始计时点不同，单个点的操作1M和0.5M比较难相遇，导致按键操作始终无法激活。故放弃此操作。
3. 第三次使用`counter_start>=1M && counter_start<1.5M`进行判断：实现消抖的基础上，还使用0.5M的间隔使得按键操作能被激活且被激活一次。实现功能！

本次实验让我基本掌握了 Verilog 的语法和 Quartus II 软件的用法，为以后进行更复杂的设计打下了基础。

