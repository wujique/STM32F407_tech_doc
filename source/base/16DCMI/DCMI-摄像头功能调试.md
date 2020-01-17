# **DCMI-摄像头功能调试**
>**够用的硬件**
**能用的代码**
**实用的教程**
>屋脊雀工作室编撰 -20190101
愿景：做一套能用的开源嵌入式驱动（非LINUX）
官网：www.wujique.com
github: https://github.com/wujique/stm32f407
淘宝：https://shop316863092.taobao.com/?spm=2013.1.1000126.2.3a8f4e6eb3rBdf
技术支持邮箱：code@wujique.com、github@wujique.com
资料下载：https://pan.baidu.com/s/12o0Vh4Tv4z_O8qh49JwLjg
QQ群：767214262
---

STM32F407芯片带有DCMI接口，在我们的核心板上已经将接口用18PIN的FPC座子引出。
这个接口可以接我们的OV2640接口。
本节我们开始调试摄像头。

## DCMI
DCMI接口是ST自己定义的接口。
>Digital camera interface (DCMI)，是意法半导体公司产品STM32F4xx系列芯片的快速摄像头接口。
通过HSVNC VSVNC PIXCLX和8到14位的数据接口D[0:13]完成控制。
#### 简介
DCMI数字摄像头接口是一个同步并行接口，能接收外部8位、10位、12位或14位CMOS摄像头模块发出的高速数据流。可支持不同的数据格式:YCbCr4:2:2/RGB565逐行视频和压缩数据（JPEG）。
此接口适用于黑白摄像头、X24和X5摄像头，并假定所有预处理（如调整大小）都在摄像头中执行。

#### 特性
![DCMI特性](pic/DCMI特性.jpg)

#### 引脚
数据输入一共有14根引脚。
![DCMI引脚](pic/DCMI引脚.jpg)

#### DCMI框图
从框图可见，DCMI支持DMA传输。
![DCMI框图](pic/DCMI框图.jpg)

## OV2640
OV2640 是 OV（OmniVision）公司生产的一颗 1/4 寸的 CMOS UXGA（1632*1232）图像传感器。该传感器体积小、工作电压低，提供单片 UXGA 摄像头和影像处理器的所有功能。通过 SCCB 总线控制，可以输出整帧、子采样、缩放和取窗口等方式的各种分辨率 8/10 位影像数据。该产品 UXGA 图像最高达到 15 帧/秒（SVGA 可达 30 帧， CIF 可达 60 帧）。用户可以完全控制图像质量、数据格式和传输方式。所有图像处理功能过程包括伽玛曲线、白平衡、对比度、色度等都可以通过 SCCB 接口编程。 OmmiVision 图像传感器应用独有的传感器技术，通过减少或消除光学或电子缺陷如固定图案噪声、拖尾、浮散等，提高图像质量，得到清晰的稳定的彩色图像。

#### 主要特性
1. 总共1632*1232像素，最大输出尺寸UXGA(1600*1200)，即200W像素。
2. 模拟功能：模拟放大、增益控制、通道平衡、平衡控制。
3. 10位ID转换。
4. 自带数字信号处理器，主要功能：边沿锐化、颜色空间转换、色相和饱和度控制、降噪等。
5. 自带压缩引擎。

#### 接口
* SCCB接口

SCCB接口控制图像传感器芯片的运行。SCCB接口相当于I2C。

* 数字视频接口

OV2640拥有一个10位数字视频接口，接口支持8位接法。

#### 图像格式
OV2640支持多种图像格式：
UXGA 1600\*1200像素。
SXGA 1280\*1024
WXGA+ 1440\*900
XVGA 1280\*960
WXGA 1280\*800
XGA 1024\*768
SVGA 800\*600
VGA 640\*480
CIF 352\*288
WQVGA 400\*240
QCIF 176\*144
QQVGA 160\*120

#### 主要操作
OV2640支持传感器窗口设置、图像尺寸设置、图像窗口设置和图像输出大小设置。
* 传感器窗口设置
设置整个传感器区域感光部分。开窗范围从2*2~1632*1220。开窗必须大于图像尺寸设置。
* 图像尺寸设置
DSP输出图像的尺寸。
* 图像窗口设置
在DSP输出的图像上开窗，窗口必须小于等于DSP输出图像的尺寸。
* 图像输出大小设置
最终输出图像的尺寸。这个图像是DSP将图像窗口中的图像**缩放**而得。

#### 使用
OV2640设置不简单，文档也复杂，我们使用OV260，其实是将ST官方的例程移植到我们的硬件上而已。

## 接口原理图
![DCMI接口](pic/接口图.jpg)

>1. SCL和SDA是摄像头控制信号，接到CPU硬件I2C上，不使用模拟I2C
>2. RESET和PWDN信号是摄像头复位和上电信号，可以不接。
>3. XCLK是时钟，屋脊雀OV2640摄像头模块不带晶振，因此需要STM32提供时钟。
>4. 17/18脚供电和地。
>5. 余下的就是DCMI信号。**其中数据线只使用8根**。

## 编码与调试
标准库提供了DCMI例程。
在STM32F4xx_DSP_StdPeriph_Lib_V1.8.0\Project\STM32F4xx_StdPeriph_Examples\DCMI\DCMI_CameraExample目录的readme.txt文件中有详细说明。
>This example shows how to use the DCMI to control the OV9655 or OV2640 Camera module
mounted on STM324xG-EVAL or STM32437I-EVAL evaluation boards.
...

从readme中可知：
>1. 使用SCCB接口配置OV9655，SCCB是一种类似I2C的协议。
>2. 将图像显示到LCD，使用了DMA。是为了释放CPU，以便执行其他任务。
>3. OV9655可以做到15帧。
>4. 相机亮度可以微调（分析后可知，是通过一个ADC采样，然后去设置摄像头）
>5. QQVGA(160x120) or QVGA(320x240)两种格式。

#### 分析例程代码
从main入手分析。

* 主流程分析

第5行，调用OV2640_HW_Init初始化硬件。
第8/9行，读ID(**基本上，调试外设，都是先获取ID**)。
第11行，Camera_Config。这个函数在camera_api.c，没有实际功能，只是根据摄像头型号以及ImageFormat（图像格式）调用对应摄像头的驱动函数配置摄像头。
第14行，启动DMA。
第15行，开启DCMI
第18行，设置LCD显示区域
第19/20行，准备写数据到LCD。**在这里直接操作LCD寄存器，是一个很粗暴的做法。**
然后就进入while(1)了，在while中，根据ADC采样，调节摄像头亮度。

```c {.line-numbers}
  /* ADC configuration */
  ADC_Config();

  /* Initializes the DCMI interface (I2C and GPIO) used to configure the camera */
  OV2640_HW_Init();

  /* Read the OV9655/OV2640 Manufacturer identifier */
  OV9655_ReadID(&OV9655_Camera_ID);
  OV2640_ReadID(&OV2640_Camera_ID);
  。。。 一些显示LCD提示语
  Camera_Config();
  。。。
  /* Enable DMA2 stream 1 and DCMI interface then start image capture */
  DMA_Cmd(DMA2_Stream1, ENABLE);
  DCMI_Cmd(ENABLE);

  /* LCD Display window */
  LCD_SetDisplayWindow(179, 239, 120, 160);
  LCD_WriteReg(LCD_REG_3, 0x1038);
  LCD_WriteRAM_Prepare();

  while(1)
  {
    /* Get the last ADC3 conversion result data */
    uhADCVal = ADC_GetConversionValue(ADC3);

    /* Change the Brightness of camera using "Brightness Adjustment" register:
       For OV9655 camera Brightness can be positively (0x01 ~ 0x7F)
             and negatively (0x80 ~ 0xFF) adjusted
       For OV2640 camera Brightness can be positively (0x20 ~ 0x40)
             and negatively (0 ~ 0x20) adjusted */
    if(Camera == OV9655_CAMERA)
    {
      OV9655_BrightnessConfig(uhADCVal);
    }
    if(Camera == OV2640_CAMERA)
    {
      OV2640_BrightnessConfig(uhADCVal/2);
    }
  }
```

* OV2640驱动

例程中的dcmi_ov2640.c就是OV2640驱动。
开头就是几个长数组，这些数组是配置到OV2640的，不同数组配置不同的图像格式。
话说，这么长的数组，如果没有原厂提供，根据资料你自己能写好吗？
所以我认为这个跟LCD初始化一样，我们不用去分析，只要用就行了。
```c
const char OV2640_QQVGA[][2]
const unsigned char OV2640_QVGA[][2]
const unsigned char OV2640_JPEG_INIT[][2]
const unsigned char OV2640_YUV422[][2]
const unsigned char OV2640_JPEG[][2]
const unsigned char OV2640_160x120_JPEG[][2]
const unsigned char OV2640_176x144_JPEG[][2]
const unsigned char OV2640_320x240_JPEG[][2]
const unsigned char OV2640_352x288_JPEG[][2]
```

接下来是HW初始化函数，将对应的IO初始化为DCMI和I2C功能。
```c
void OV2640_HW_Init(void)
```

接下来的一个重要函数就是OV2640初始化，完成DCMI和DMA的初始化。
```c {.line-numbers}
/**
  * @brief  Configures DCMI/DMA to capture image from the OV2640 camera.
  * @param  ImageFormat: Image format BMP or JPEG
  * @param  BMPImageSize: BMP Image size
  * @retval None
  */
void OV2640_Init(ImageFormat_TypeDef ImageFormat)
{
  DCMI_InitTypeDef DCMI_InitStructure;
  DMA_InitTypeDef  DMA_InitStructure;

  /*** Configures the DCMI to interface with the OV2640 camera module ***/
  /* Enable DCMI clock */
  RCC_AHB2PeriphClockCmd(RCC_AHB2Periph_DCMI, ENABLE);

  /* DCMI configuration */
  DCMI_InitStructure.DCMI_CaptureMode = DCMI_CaptureMode_Continuous;
  DCMI_InitStructure.DCMI_SynchroMode = DCMI_SynchroMode_Hardware;
  DCMI_InitStructure.DCMI_PCKPolarity = DCMI_PCKPolarity_Rising;
  DCMI_InitStructure.DCMI_VSPolarity = DCMI_VSPolarity_Low;
  DCMI_InitStructure.DCMI_HSPolarity = DCMI_HSPolarity_Low;
  DCMI_InitStructure.DCMI_CaptureRate = DCMI_CaptureRate_All_Frame;
  DCMI_InitStructure.DCMI_ExtendedDataMode = DCMI_ExtendedDataMode_8b;

  /* Configures the DMA2 to transfer Data from DCMI */
  /* Enable DMA2 clock */
  RCC_AHB1PeriphClockCmd(RCC_AHB1Periph_DMA2, ENABLE);

  /* DMA2 Stream1 Configuration */  
  DMA_DeInit(DMA2_Stream1);

  DMA_InitStructure.DMA_Channel = DMA_Channel_1;  
  DMA_InitStructure.DMA_PeripheralBaseAddr = DCMI_DR_ADDRESS;
  DMA_InitStructure.DMA_Memory0BaseAddr = FSMC_LCD_ADDRESS;
  DMA_InitStructure.DMA_DIR = DMA_DIR_PeripheralToMemory;
  DMA_InitStructure.DMA_BufferSize = 1;
  DMA_InitStructure.DMA_PeripheralInc = DMA_PeripheralInc_Disable;
  DMA_InitStructure.DMA_MemoryInc = DMA_MemoryInc_Disable;
  DMA_InitStructure.DMA_PeripheralDataSize = DMA_PeripheralDataSize_Word;
  DMA_InitStructure.DMA_MemoryDataSize = DMA_MemoryDataSize_HalfWord;
  DMA_InitStructure.DMA_Mode = DMA_Mode_Circular;
  DMA_InitStructure.DMA_Priority = DMA_Priority_High;
  DMA_InitStructure.DMA_FIFOMode = DMA_FIFOMode_Enable;
  DMA_InitStructure.DMA_FIFOThreshold = DMA_FIFOThreshold_Full;
  DMA_InitStructure.DMA_MemoryBurst = DMA_MemoryBurst_Single;
  DMA_InitStructure.DMA_PeripheralBurst = DMA_PeripheralBurst_Single;

  switch(ImageFormat)
  {
    case BMP_QQVGA:
    {
      /* DCMI configuration */
      DCMI_InitStructure.DCMI_VSPolarity = DCMI_VSPolarity_High;
      DCMI_Init(&DCMI_InitStructure);

      /* DMA2 IRQ channel Configuration */
      DMA_Init(DMA2_Stream1, &DMA_InitStructure);
      break;
    }
    case BMP_QVGA:
    {
      /* DCMI configuration */
      DCMI_Init(&DCMI_InitStructure);

      /* DMA2 IRQ channel Configuration */
      DMA_Init(DMA2_Stream1, &DMA_InitStructure);
      break;
    }
     default:
    {
      /* DCMI configuration */
      DCMI_InitStructure.DCMI_VSPolarity = DCMI_VSPolarity_High;
      DCMI_Init(&DCMI_InitStructure);

      /* DMA2 IRQ channel Configuration */
      DMA_Init(DMA2_Stream1, &DMA_InitStructure);
      break;
    }
  }
}
```
32~46行是DMA的初始化，前面已经说过DMA，本处就不累赘了。
17~23是DCMI的初始化。
>17行，连续模式
18行，硬件同步
19行，PCK上升沿有效
20行，VSYNC低电平有效
21行，HSYNC低电平有效
22行，全帧捕获
23行，8位数据模式。

48行~77行，这么多代码完成了什么功能？
**这么多代码只做了一件事，就是当QQVGA模式时，VSYNC用高电平有效。QVGA才用低电平有效**。

剩下的其他函数就是将前面说的数组配置到OV2640，没什么其他重要功能了。

这样看来，OV2640用起来也不算难。

#### 移植
将dcmi_ov9655.c、dcmi_ov9655.h、dcmi_ov2640.c、dcmi_ov2640.h、camera_api.c、camera_api.h拷贝到我们工程的board_dev目录，添加到SI跟MDK工程，开始移植。
程序结构上做以下修改：
>1. 将DCMI相关函数放到mcu_dcmi.c内。
>2. 将SCCB相关函数放到mcu_i2c.c内。
>3. 原来main函数中的测试代码放到camera_api.c中。

差异：
>摄像头上没有晶振，需要使用STM32的MCO1输出时钟给摄像头使用。

0. 初始化

创建了一个初始化函数，硬件相关的初始化都放在这个函数内。
官方例程放在OV9655和OV2640内，两套，不合理。
这里所谓的初始化，只是初始化摄像头接口。
跟你用什么摄像头，无关。
MCO1_Init就是初始化MCO管脚，输出时钟给摄像头。
```c
s32 dev_camera_init(void)
{
	/* camera xclk use the MCO1 */
	MCO1_Init();
	DCMI_PWDN_RESET_Init();

	/* Initializes the DCMI interface (I2C and GPIO) used to configure the camera */
	BUS_DCMI_HW_Init();

	SCCB_GPIO_Config();
	return 0;
}
```

1. 修改I2C配置，调试到能读到摄像头ID

也就是修改
```c
void SCCB_GPIO_Config(void)
uint8_t bus_sccb_writereg(uint8_t DeviceAddr, uint16_t Addr, uint8_t Data)
uint8_t bus_sccb_readreg(uint8_t DeviceAddr, uint16_t Addr)
```
查参考手册，我们用的硬件I2C是I2C2，把上面3个函数中的控制器全部改为I2C2？----这方法不好，应该全部改为宏定义。宏定义改起来方便。

调试信息，能正确读到ID。
>---hello world!-----
init finish!
read reg:9341
lcd init ok!
camera test....
OV9655 Camera ID 0x96
Camera_Config...
test camera!
test camera!
test camera!
test camera!

2. 移植DCMI跟DMA功能

>修改DCMI相关IO。
修改LCD 显示RAM的地址，在LCD章节，我们说过这个地址为什么是0x6C010000。

```c
#define DCMI_DR_ADDRESS       0x50050028
#define FSMC_LCD_ADDRESS      0x6C010000
```
3. 摄像头数据采集方向和LCD扫描方向要一致。

LCD驱动器ILI9341如果配置为横屏模式，则要先左右，后上下；
先左还是先右，先上还是先下，图像会不一样。
如果配置为竖屏，则需要使用后面四种扫描方向，也就是先上下，后左右。
简单的说，就是先扫描320像素长边，再扫描240像素的短边。

```c
#define L2R_U2D  (0) //从左到右,从上到下
#define L2R_D2U  (0 + UD_BIT_MASK)//从左到右,从下到上
#define R2L_U2D  (0 + LR_BIT_MASK) //从右到左,从上到下
#define R2L_D2U  (0 + UD_BIT_MASK + LR_BIT_MASK) //从右到左,从下到上

#define U2D_L2R  (LRUD_BIT_MASK)//从上到下,从左到右
#define U2D_R2L  (LRUD_BIT_MASK + LR_BIT_MASK) //从上到下,从右到左
#define D2U_L2R  (LRUD_BIT_MASK + UD_BIT_MASK) //从下到上,从左到右
#define D2U_R2L  (LRUD_BIT_MASK + UD_BIT_MASK+ LR_BIT_MASK) //从下到上,从右到左	 
```
摄像头调试过程，参考官方例程，很快就调通了。

4. 代码结构调整

最终的代码，我们对原厂的例程进行了部分的重新封装和调整。
使得代码架构层次清晰，模块化更好。

## 总结

无

---
end
---
