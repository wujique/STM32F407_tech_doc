# **FreeRtos移植**
>**够用的硬件**
>
>**能用的代码**
>
>**实用的教程**
>
>屋脊雀工作室编撰 -20190101
>
>愿景：做一套能用的开源嵌入式驱动（非LINUX）
>
>官网：www.wujique.com
>
>github: https://github.com/wujique/stm32f407
>
>淘宝：https://shop316863092.taobao.com/?spm=2013.1.1000126.2.3a8f4e6eb3rBdf
>
>技术支持邮箱：code@wujique.com、github@wujique.com
>
>资料下载：https://pan.baidu.com/s/12o0Vh4Tv4z_O8qh49JwLjg
>
>QQ群：767214262
---

**待补充完善**

很多同学可能从一开始就盼着这天了。
其实我想说的是，什么操作系统都是纸老虎，等你学会了，就知道，不过如此。
学嵌入式的路程，基本上是：裸奔-RTOS-LINUX。
我们现在就开始结束裸奔。
怎么学RTOS呢？我觉得按照下面四个步骤学习，效果不错。
1. 了解RTOS的概念。
2. 找一个合适的RTOS了解概况。
3. 用RTOS跑起来。
4. 再回头研究RTOS的实现。

## RTOS

什么是RTOS？
RTOS = **Real Time Operating System**，实时操作系统。
#### 基本概念
操作系统，我们电脑上的WINDOW就是一个操作系统。
与WINDOWS对应的，就是我们经常说的linux。但是我们经常有一个误解，其实linux不是一个操作系统，而是**操作系统内核**。
基于linux内核的操作系统很多，各种LINUX发行版都是，例如红帽，ubuntu等。
安卓系统也是基于linux内核的。

那实时操作系统和普通电脑手机上的操作系统有什么区别呢？
先看看百度百科定义：
>实时操作系统（RTOS）是指当外界事件或数据产生时，能够接受并以足够快的速度予以处理，其处理的结果又能在规定的时间之内来控制生产过程或对处理系统做出快速响应，调度一切可利用的资源完成实时任务，并控制所有实时任务协调一致运行的操作系统。提供及时响应和高可靠性是其主要特点。

上面是定义，很多同学学习时，都会认为实时操作系统就是一个**响应很快**的操作系统。
这个是片面的。实时，确实比WIDOW这些PC操作系统快，但快，不是我们的指标。
实时，更准确的描述是，响应时间确定，稳定。
如果一个响应，说是十秒钟之内会响应，就永远不会11秒才响应，这才叫实时。
WINDOW就不是一个实时系统，很多操作经常会被其他应用卡住，甚至死机。

说到这里，都是书面内容，我们还是不明白RTOS到底是什么鬼。
RTOS通常有以下特点：
>小，通常只有几个文件，代码量也很小，需要的内存也很小，通常几K就能跑起来。
如果功能不复杂，任务不多，在一个增强型8051都能运行起来。

还是不懂什么是RTOS，怎么办？

#### 大循环任务架构

我们要一个RTOS来做什么？
我们用RTOS的一个最重要功能，是任务管理。
经过前面的实验，我们知道一个裸奔的嵌入式程序，就是一个大循环，一个while(1)里面的大循环。
所有放在while(1)循环里面的任务，都不能卡死。只要一个任务卡死，其他任务就都不执行了。
为了达到大循环的目的，我们介绍一个程序架构方法：**步骤拆分**。不知道大家还记得吗？
代码框架如下：
```c
void n_task(void)
{
	switch(步骤)
	{
		case 步骤1:
			break;
		case 步骤2:
			break;
		case 步骤3:
			break;
	}
}

void b_task(void)
{
	switch(步骤)
	{
		case 步骤1:
			break;
		case 步骤2:
			break;
		case 步骤3:
			break;
	}
}

void app_task(void)
{
	switch(步骤)
	{
		case 步骤1:
			break;
		case 步骤2:
			break;
		case 步骤3:
			break;
	}
}

void main(void)
{
	初始化();
	while(1)
	{
		/*  底层驱动 */
		n_task();
		b_task();

		/* 应用流程 */
		app_task();
	}
}
```

但是，我要说的是：**大循环**，是有点反人类的。每一个任务（驱动流程），都要去管别人，自己怎么执行的，也会受到别人影响。要很小心的设计步骤，才能解决所有任务的运行矛盾。
如果，一个程序，有复杂的APP，跟底层的矛盾，就更难解决了。而且，复杂的应用层，很难改为大循环。

#### 小循环代码架构

与大循环对应的，就是**小循环**。
什么是小循环？
每个功能，都写成一个while(1)，就是小循环。
在想用while(1)的地方，就用while(1)，就是小循环。
例如，我们在按键扫描驱动中，在scan函数中用while(1)一直扫描。
代码框架如下：
```c
void n_task(void)
{
	while(1)
	{
		switch(步骤)
		{
			case 步骤1:
				break;
			case 步骤2:
				break;
			case 步骤3:
				break;
                }
	}
}

void b_task(void)
{
	while(1)
	{
		while(1)
		{
			if(条件符合)
				break;
		}
		while(1)
		{
			if(条件符合)
				break;
		}
		while(1)
		{
			if(条件符合)
				break;
		}
	}
}

void app_task(void)
{
	while(1)
	{
		while(1)
		{
			if(条件符合)
				break;
		}
		while(1)
		{
			if(条件符合)
				break;
			else
			{
				while(1)
				{
					等某种状态
						break;
				}
			}
		}
	}
}

void main(void)
{
	初始化();
	while(1)
	{
		/*  底层驱动 */
		n_task();
		b_task();

		/* 应用流程 */
		app_task();
	}
}
```
这样每个驱动都是一个死循环，程序怎么跑呢？肯定跑不起来。
嵌入式OS，就是让各个死循环跑起来的代码。

每个TASK我们就叫做任务，OS，就是实现多任务运行。
在每个task里面，可以随便按照你自己的需求写多个小循环。
#### 基本原理
RTOS多任务是怎么实现的呢？
如果你学过汇编，可能就容易理解。
首先，如何打断代码执行？中断，对，就是中断。
每一个RTOS都会有一个定时器中断，我们叫做系统滴答，例如，系统滴答是2ms，那么就是2ms就会产生一次定时中断，这个中断打断了正在执行的程序。在这个定时中运行RTOS的管理代码，这时，控制权在RTOS手上，想怎么搞就怎么搞。普通的中断一般会返回原来的程序位置运行。
系统滴答中断，故意不回到原来的地方，而是切到其他地方运行，那么，就完成了一次任务切换。
为了回到上次的任务，就需要将上次的运行状态记住。等需要的时候，就可以继续执行。


任务管理，是RTOS的最基本功能。为了支持多任务，一个RTOS还需要其他功能，例如任务间通信的各种手段：互斥锁，邮箱，信号等。

## FreeRtos
很早之前，说到RTOS，都是说UCOS。
其他，RTOS有很多很多：UCOS、freertos、rtems。。。。
以前UCOS流行，因为他说开源的。但是，其实UCOS开源不免费，商业使用是需要授权费的。
而freertos，是免费的。现在物联网兴起，很多zigbee和wifi芯片都选择免费的freertos。所以这几年freertos爆发了。
我们的代码都是免费的，我们当然选择免费的freertos。

百度百科
>FreeRTOS是一个迷你的实时操作系统内核。作为一个轻量级的操作系统，功能包括：任务管理、时间管理、信号量、消息队列、内存管理、记录功能、软件定时器、协程等，可基本满足较小系统的需要。
由于RTOS需占用一定的系统资源(尤其是RAM资源)，只有μC/OS-II、embOS、salvo、FreeRTOS等少数实时操作系统能在小RAM单片机上运行。相对μC/OS-II、embOS等商业操作系统，FreeRTOS操作系统是完全免费的操作系统，具有源码公开、可移植、可裁减、调度策略灵活的特点，可以方便地移植到各种单片机上运行，其最新版本为10.0.1版。

官网：https://www.freertos.org/

## 移植FreeRtos
如何移植？
规范的方法：
1. 看freertos的程序包，参考他的移植。一个程序要推广，官方会移植到很多平台。
freertos的demo，有170多个芯片的范例，总有一个适合你。
2. 看平台的例程，我们可以从ST官网找到freertos的移植范例。

我们更多时候，是网络搜索别人移植好的。

下面我们不管参考哪个例程，只说说移植需要考虑的一些细节问题。

1. 移植前，把所有freertos文件增加ft前缀。

我们已经移植了LWIP，有一些源码文件和freertos重名。
但是我的程序是在所有硬件都编写了驱动之后才移植freertos，代码包含了LWIP。
>在移植过程出现了很多错误，例如
```c
..\Utilities\FreeRTOS\Source\timers.c(76): error:  #20: identifier
"TimerCallbackFunction_t" is undefined
  	TimerCallbackFunction_t	pxCallbackFunction;
		/*<< The function that will be called when the timer expires. */
..\Utilities\FreeRTOS\Source\timers.c(104): error:  #20: identifier
"PendedFunction_t" is undefined
  	PendedFunction_t	pxCallbackFunction;
		/* << The callback function to execute. */
..\Utilities\FreeRTOS\Source\timers.c(220): error:  #20: identifier
"TimerCallbackFunction_t" is undefined
  									TimerCallbackFunction_t pxCallbackFunction,
..\Utilities\FreeRTOS\Source\timers.c(279): error:  #20: identifier
"TimerHandle_t" is undefined
```
但是这些定义命名全部都在timers.h里面有定义。
点开工程timer.c，查看里面的头文件，你会发现竟然包含的是LWIP的头文件。
![头文件包含错误](pic/1.jpg)
为了避免更多麻烦，决定在freertos所有头文件加上ft前缀
。

还有就是我们选择屏蔽下面代码FreeRTOSConfig.h

```c
//#define vPortSVCHandler SVC_Handler
//#define xPortPendSVHandler PendSV_Handler
//#define xPortSysTickHandler SysTick_Handler
```

而是直接将这些系统用的函数塞到stm32f4xx_it.c文件中的中断内。
1 保持所有中断响应都在这个文件内处理
2 我们要用与freertos系统无关的延时函数Delay。
3 freeRTOS通常会将systick定义10毫秒，比较粗，我们可能需要将系统滴答精确到毫秒。

同时在FreeRTOSConfig.h文件最后定义，也就是内存分配用我们自己的，不用freertos提供的。
```c
#define pvPortMalloc wjq_malloc
#define vPortFree	wjq_free
```

尽快启动任务，这样就可以将startup_stm32f40_41xxx里面定义的堆跟栈设置为最少，甚至设置为0

## 总结

---
end
---
