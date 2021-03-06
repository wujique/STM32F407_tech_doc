# DAC SOUND驱动改造--播放WAV文件
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

前面播放WAV音频和I2S录音两个小节，我们接触了一种叫做中间件的程序。
我可以可以再总结一下：
>所谓的中间件，通常是实现一种功能的抽象接口。这一层代码，对应用屏蔽了硬件实现，只提供功能接口。
例如：LCD中间件，GUI也可以算中间件，应用层主要调用LCD显示接口，就可以在各种LCD伤显示内容。
那么，语音播放中间件，就是APP播放音乐，可以在多种硬件声音设备上播放。

前面章节我们已经实现了WM8978播放，我们硬件正好还有一个DAC SOUND的设备。
怎么样修改DAC SOUND代码，让其在语音播放中间件下也能工作？

## 框架设计

不用多想就可以知道，既然都是在SOUND中间件下工作的硬件，那么驱动，肯定应该差不多。
我们已经完成了WM8978的驱动设计，DAC SOUND按其修改，肯定是最合理的。

#### DAC SOUND
前面我们做的DAC SOUND功能：
>启动一个定时器，按照音频采样率，定时从数组读取样点，通过DAC输出。

DAC SOUND和WM8978的一个最大不同点就是：**它是单声道的**。

#### 双缓冲
如果要根据I2S驱动修改DAC SOUND，只需要简单的将数组改为双缓冲。
语音中间件只需要将原来打开WM8978设备改为DAC设备就可以了。
中间件数据处理部分代码都不需要修改。

#### 代码说明

初始化函数，没变，也就是初始化IO口。
```c {.line-numbers}
/*

	DAC 播放声音，固定播放8K单声道16BIT的音源。

*/

u16 *DacSoundSampleP0;
u16 *DacSoundSampleP1;
u16 *DacSoundCrBufP;//当前使用的BUF

u32 DacSoundSampleBufSize;
u32 DacSoundSampleIndex;

s32 dev_dacsound_init(void)
{
	GPIO_InitTypeDef GPIO_InitStructure;

	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_5;
        GPIO_InitStructure.GPIO_Mode = GPIO_Mode_OUT;//---模拟模式
        GPIO_InitStructure.GPIO_PuPd = GPIO_PuPd_UP;//---下拉
        GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
        GPIO_Init(GPIOA, &GPIO_InitStructure);//---初始化 GPIO

	GPIO_ResetBits(GPIOA, GPIO_Pin_5);

	return 0;
}
```

打开播放函数，除了初始化DAC和定时器，还需要初始化两个缓冲的指针。
```c
s32 dev_dacsound_open(void)
{
	mcu_dac_open();
	mcu_tim3_init();
	DacSoundSampleIndex = 0;
	DacSoundCrBufP = DacSoundSampleP0;

	return 0;
}
```
空函数，暂时不做太复杂，只播放8K采样率的WAV。
```c
/**
 *@brief:      dev_dacsound_dataformat
 *@details:       设置播放配置，DAC播放固定支持8K 16BIT 单声-
                  道
 *@param[in]   u32 Freq     
               u8 Standard  
               u8 Format    
 *@param[out]  无
 *@retval:     
 */
s32 dev_dacsound_dataformat(u32 Freq, u8 Standard, u8 Format)
{
	return 0;
}
```
初始化缓冲指针，模拟I2S DMA配置双缓冲。
```c
/**
 *@brief:      dev_dacsound_setbuf
 *@details:    设置播放缓冲
 *@param[in]   u16 *buffer0  
               u16 *buffer1  
               u32 len       
 *@param[out]  无
 *@retval:     
 */
s32 dev_dacsound_setbuf(u16 *buffer0,u16 *buffer1,u32 len)
{
	DacSoundSampleP0 = buffer0;
	DacSoundSampleP1 = buffer1;
	DacSoundSampleBufSize = len;

	return 0;
}
```
dev_dacsound_transfer函数，启动播放，也是模拟I2S的函数
```c
/**
 *@brief:      dev_dacsound_transfer
 *@details:    启动或停止DAC播放
 *@param[in]   u8 sta  
 *@param[out]  无
 *@retval:     
 */
s32 dev_dacsound_transfer(u8 sta)
{
	if(sta == 1)
	{
		/*打开定时器，启动播放*/
		DACSOUND_DEBUG(LOG_DEBUG, "dac sound play\r\n");
		mcu_tim3_start();
	}
	else
	{
		/*停止定时器*/
		mcu_tim3_stop();
	}

	return 0;
}
```
停止播放
```c
s32 dev_dacsound_close(void)
{
	dev_dacsound_init();
	return 0;
}
```

定时器中断函数，在这个函数内将缓冲的数据通过DAC输出。
流程跟原来基本一致。
需要修改的是取数据的方法。
```c
/**
 *@brief:      dev_dacsound_timerinit
 *@details:    在定时器中断中调用，每125US输出一个DAC数据
 			   能不能改为DMA？
 *@param[in]   void  
 *@param[out]  无
 *@retval:     
 */
s32 dev_dacsound_timerinit(void)
{
    s16 data = 0;
    u16 tmp;

	if(DacSoundSampleIndex >= DacSoundSampleBufSize)
	{
		if(DacSoundCrBufP == DacSoundSampleP0)
		{
			DacSoundCrBufP = DacSoundSampleP1;
			fun_sound_set_free_buf(0);
		}
		else
		{
			DacSoundCrBufP = DacSoundSampleP0;
			fun_sound_set_free_buf(1);
		}
		DacSoundSampleIndex = 0;

	}

	/*要注意，读到的数据是S16，正负值*/
	data = *(DacSoundCrBufP + DacSoundSampleIndex);
	/*
		先压缩，也就是减少音量，在负数时候压缩（除）
		压缩方向时中位值，如果先将负数调整为正数（抬高直流电平），
		压缩方向就会变成音频的最低值，音效会失真。
	*/
	data = data/(16+30);//12位DAC，再加上音量设置，
	/*再调整中位值(直流电平)，因为音频数据有负数，DAC输出没有负数*/
	tmp = (data+0x800);
	mcu_dac_output(tmp);

	DacSoundSampleIndex++;
	return 0;
}
```

## 中间件修改

在 int fun_sound_play(char *name, char *dev)内，原来只有WM8978设备，
现添加DAC SOUND设备。
```c

	if(0 == strcmp(dev, "wm8978"))
	{
		dev_wm8978_open();
		dev_wm8978_dataformat(wav_header->nSamplesPerSec,
			WM8978_I2S_Phillips, format);
		mcu_i2s_dma_init(SoundBufP[0], SoundBufP[1], SoundBufSize);
		SoundDevType = SOUND_DEV_2CH;
		dev_wm8978_transfer(1);//启动I2S传输
	}
	else if(0 == strcmp(dev, "dacsound"))
	{
		dev_dacsound_open();
		dev_dacsound_dataformat(wav_header->nSamplesPerSec,
			WM8978_I2S_Phillips, format);
		dev_dacsound_setbuf(SoundBufP[0], SoundBufP[1], SoundBufSize);
		SoundDevType = SOUND_DEV_1CH;
		dev_dacsound_transfer(1);
	}

```

fun_sound_stop函数同样添加
```c
if(SoundDevType == SOUND_DEV_2CH)
{
  dev_wm8978_transfer(0);
}
else if(SoundDevType == SOUND_DEV_1CH)
{
  dev_dacsound_transfer(0);
  dev_dacsound_close();
}
```

## 应用
只需要在播放语音的时候指定DAC设备。
```c
/**
 *@brief:      fun_sound_test
 *@details:    测试播放
 *@param[in]   void  
 *@param[out]  无
 *@retval:     
 */
void fun_sound_test(void)
{
	SOUND_DEBUG(LOG_DEBUG, "play sound\r\n");
	fun_sound_play("1:/mono_16bit_8k.wav", "dacsound");		

}
```

## 总结
简单吧？确实简单。
这么简单的原因是，我们良好的架构设计。
但愿我们能教会大家写好代码。

---
end
---
