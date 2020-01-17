# VSPI控制COG LCD & I2C控制OLED
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

上一节我们已经点亮了COG LCD跟OLED LCD，用的是外扩SPI。
在核心板上的外扩接口中，除了硬件SPI外，还有多个IO口，可以用来模拟SPI。
还有一个I2C，正好可以用来控制I2C接口的OLED LCD。
就让我们来完善我们的LCD驱动，让它支持更多方式吧。

## 驱动
驱动上一节已经实现，不需要修改。
## 底层接口
上一节我们已经定义了一些函数，用于LCD驱动操作硬件。
如下：
```c
void bus_seriallcd_IO_init(void)
s32 bus_seriallcd_bl(u8 sta)
s32 bus_seriallcd_init()
s32 bus_seriallcd_open()
s32 bus_seriallcd_close()
s32 bus_seriallcd_write_data(u8 *data, u16 len)
s32 bus_seriallcd_write_cmd(u8 cmd)
```
其中bus_seriallcd_IO_init内部接口。
现在我们要添加模拟SPI跟I2C。因此将这个串行LCD接口抽象一个结构体，
```c
typedef struct  
{
	char * name;

	s32 (*init)(void);
	s32 (*open)(void);
	s32 (*close)(void);
	s32 (*writedata)(u8 *data, u16 len);
	s32 (*writecmd)(u8 cmd);
	s32 (*bl)(u8 sta);
}_lcd_bus;
```
也即是说，一个LCD接口，只需要提供这些功能即可。
然后定义三个LCD接口，并且实现他们的功能函数，以下分别是I2C、VSPI、SPI。
函数具体实现请看代码。
```c
_lcd_bus BusSerialLcdVI2C={
		.name = "BusSerivaLcdVI2C",
		.init =bus_seriallcd_vi2c_init,
		.open =bus_seriallcd_vi2c_open,
		.close =bus_seriallcd_vi2c_close,
		.writedata =bus_seriallcd_vi2c_write_data,
		.writecmd =bus_seriallcd_vi2c_write_cmd,
		.bl =bus_seriallcd_vi2c_bl,				
};

_lcd_bus BusSerialLcdVSpi={
		.name = "BusSerivaLcdVSpi",
		.init =bus_seriallcd_vspi_init,
		.open =bus_seriallcd_vspi_open,
		.close =bus_seriallcd_vspi_close,
		.writedata =bus_seriallcd_vspi_write_data,
		.writecmd =bus_seriallcd_vspi_write_cmd,
		.bl =bus_seriallcd_vspi_bl,				
};

_lcd_bus BusSerialLcdSpi={
		.name = "BusSerivaLcdSpi",
		.init =bus_seriallcd_spi_init,
		.open =bus_seriallcd_spi_open,
		.close =bus_seriallcd_spi_close,
		.writedata =bus_seriallcd_spi_write_data,
		.writecmd =bus_seriallcd_spi_write_cmd,
		.bl =bus_seriallcd_spi_bl,				
};
```
那我们用哪个接口呢？
定义一个接口指针，要用哪个接口就赋值对应结构体。

```c
_lcd_bus *LcdBusDrv = &BusSerialLcdVI2C;
```
并且修改驱动，原来直接调用函数的地方改为变量指针，例如：
```c
static s32 drv_ST7565_refresh_gram(u16 sc, u16 ec, u16 sp, u16 ep)
{
	struct _cog_drv_data *gram;
	u8 i;

	//uart_printf("drv_ST7565_refresh:%d,%d,%d,%d\r\n", sc,ec,sp,ep);
	gram = (struct _cog_drv_data *)&LcdGram;

	LcdBusDrv->open();
    for(i=sp/8; i <= ep/8; i++)
    {
        LcdBusDrv->writecmd (0xb0+i);    //设置页地址（0~7）
        LcdBusDrv->writecmd (((sc>>4)&0x0f)+0x10);      //设置显示位置—列高地址
        LcdBusDrv->writecmd (sc&0x0f);      //设置显示位置—列低地址

         LcdBusDrv->writedata(&(gram->gram[i][sc]), ec-sc+1);

	}
	LcdBusDrv->close();

	return 0;
}

```

#### 模拟SPI

在触摸芯片章节我们已经实现了一个模拟SPI。
现在多增加一个。
```c

#define VSPI2_CS_PORT GPIOF
#define VSPI2_CS_PIN GPIO_Pin_12

#define VSPI2_CLK_PORT GPIOF
#define VSPI2_CLK_PIN GPIO_Pin_11

#define VSPI2_MOSI_PORT GPIOF
#define VSPI2_MOSI_PIN GPIO_Pin_10

#define VSPI2_MISO_PORT GPIOF
#define VSPI2_MISO_PIN GPIO_Pin_9

#define VSPI2_RCC RCC_AHB1Periph_GPIOF

DevVspiIO DevVspi2IO={
		"VSPI2",
		DEV_VSPI_2,
		-2,//未初始化

		VSPI2_RCC,
		VSPI2_CLK_PORT,
		VSPI2_CLK_PIN,

		VSPI2_RCC,
		VSPI2_MOSI_PORT,
		VSPI2_MOSI_PIN,

		VSPI2_RCC,
		VSPI2_MISO_PORT,
		VSPI2_MISO_PIN,

		VSPI2_RCC,
		VSPI2_CS_PORT,
		VSPI2_CS_PIN,
	};

```

#### I2C
I2C就用前面调好的，不需要修改。

#### 层次拆分
到此，我们实现了功能，但是，你有没有感觉，**LCD硬件接口**跟**LCD驱动**，是两层意思？
也就是说驱动分四层：
**LCD中间层**——**LCD驱动层**——**LCD硬件接口**——**对应的通信接口**。
对应：
LCD显示 —— SSD1565驱动——SPI LCD接口——SPI驱动
## 总结
由于我们程序良好的架构设计，仅仅修改了LCD硬件接口层的处理方式，就将原来使用SPI接口的LCD驱动，改成VSPI接口和I2C接口。
而且，LCD驱动可以说基本上没修改。

1. LCD驱动已经能支持我们的外扩3种接口。
请问：
  TFT 8080接口是否能用这种封装？

2. 现在的代码，同时只能支持一个硬件设备。具体用什么设备，在dev_lcd_init中选，用什么总线，通过LcdBusDrv指针选。
如果需要同时使用，怎么办？比如8080，VSPI,I2C,SPI，四个接口全部接上LCD。

**请看下一节。**

**本节例程代码，只是给大家参考，如果设计项目使用，建议用屋脊雀在github上的最新代码架构**


---
end
---
