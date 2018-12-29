#Write in front
本网页将我这一年（2018年）的电控经历积累下来的知识和经验，以及下半年进行的两次培训的内容和要点做了一个综合总结。
上半年我一直用的是标准库进行编程，但自从国庆期间用上**HAL库**之后，我被HAL库的便捷深深打动，自此基本放弃了标准库，所以本总结的内容是基于HAL库的，并结合**STM32CubeMX**软件*（因为过程有些繁琐，截图很麻烦，所以本总结并不会贴出STM32CubeMX上的相关配置过程）*。
本页主要讲解STM32一些外设的基本知识，具体的案例使用见[实例](examples.html)页面。
大家有遇到什么STM32方面的问题可以私信我的QQ，我会不定期解答大家的疑惑，并将问题和答案同步更新在[FAQ](fag.html)页面，同时相关知识点也会在本页面更新。
#GPIO
GPIO外设一共有八种模式，我比较熟悉的是这三种：**上拉输入**、**下拉输入**、**推挽输出**。上拉输入和下拉输入一般都和**外部中断**联系在一起，最典型的的应用就是读取按键的输入。推挽输出的功能是控制IO口的输出电平的高低，最常见的应用就是控制LED灯的亮灭。GPIO涉及的基本函数有以三个：
##GPIO_intro
```c
HAL_GPIO_WritePin(GPIO_TypeDef * GPIOx, uint16_t GPIO_Pin, GPIO_PinState PinState);  //直接指定引脚的电平输出，用于推挽输出

HAL_GPIO_TogglePin(GPIO_TypeDef * GPIOx, uint16_t GPIO_Pin);  //直接翻转引脚的电平输出，无需指定输出电平，用于推挽输出

HAL_GPIO_ReadPin(GPIO_TypeDef * GPIOx, uint16_t GPIO_Pin);  //读取指定引脚当前的电平高低
```
函数功能见注释。前两个函数用于推挽输出，第三个函数用于输入功能。GPIO的输入还可以与外部中断联系起来，GPIO的外部中断回调函数如下：
```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
  
  //用户自己的中断函数逻辑
  //your code here ...
}
```
这里简单说一下回调函数。HAL库的最大特点之一就是有很多的回调函数，几乎所有的中断都有对应的一种或多种回调函数。这些回调函数都被官方提前定义好了，只不过是**__weak**弱定义，我们要实现自己的回调函数逻辑只需要自己重新再定义一次这个回调函数，将它原来的弱定义覆盖即可。
##EXTI
中断的概念大家应该都了解的差不多了，所以我就略过介绍中断这一环节。**外部中断（EXTI）**是STM32众多中断中的一种。STM32一共有16组外部中断，分别对应GPIOx.1~GPIOx.15(x=A,B,C,...,H,I),也就是说序号相同的IO口触发的是同一个外部中断，但是外部中断**无法判断**是由哪个GPIO外设所触发的，所以STM32最多可以监听16个外部中断，且触发这些外部中断的IO口序号**不能相同**。
#USART
STM32中有**UART**与**USART**两种串口通信，USART比UART多出来的**S**代表的是“同步”，也就是说USART既支持同步通信，也支持异步通信，而UART只支持异步通信。但是同步通信在目前阶段的开发中几乎不会接触到，普遍用的还是它的异步通信功能。
##USART_Intro
USART的标准接线有三根：TX、RX、GND，接线的时候要注意两块单片机的TX和RX要**反接**，因为发送（TX）对应着接收（RX）。个别同学可能图方便不接GND，甚至有些场景只接TX获RX一根线，这种情况虽然也可以通信成功，但是极不稳定，很容易发送失败。
因为USART的发送和接收数据分别使用一根线，所以USART串口通信支持**全双工通信**，即可以同时进行消息的发送和接收。USART的数据接收和发送都有三种方法，分别为**中断**、**DMA**和**非中断非DMA**。我推荐使用DMA接收，尤其是数据量很大的时候。这里列出了DMA方式的接收函数和非中断非DMA方式的发送函数。
```c
HAL_UART_Transmit(UART_HandleTypeDef * huart, uint8_t * pData, uint16_t Size, uint32_t Timeout);  //串口发送函数，指定发送的字节数及其长度，同时指定发送超时时间

HAL_Receive_DMA(UART_HandleTypeDef * huart, uint8_t * pData, uint16_t Size);  //串口DMA方式接收，会引发DMA中断
```
USART的发送和接收都可以设置是否触发中断，在标准库中串口通信的发送和接收中断函数是同一个，需要用户自己根据相关的寄存器状态判断是发送还是接收，而HAL库中已经写好了相关判断，并定义好了发送中断回调函数和接收中断回调函数。USART的接收中断回调函数如下：
```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef * huart)
{
  HAL_Receive_DMA(UART_HandleTypeDef * huart, uint8_t * pData, uint16_t Size);  //继续使能串口接收中断
  
  //用户自己的中断函数逻辑
  //your code here ...
}
```
大家可以注意一下这个中断回调函数的命名方式，其实很好记忆：

|    HAL|       UART |      Rx |Cplt|Callback|
|:-------:|:-------------:|:----------:|:----:|:-----:|
|   HAL库  |串口|接收|Complete简写，接收完成|回调函数|
大家如果看过HAL库的手册或者粗粗翻看过HAL库的头文件的话，会发现HAL库的回调函数特别丰富，不仅有完成回调函数，还有完成一半的回调函数，还有阻塞状态的回调函数等等，有兴趣的朋友可以自己去探索。
##printf
另外这里介绍一下怎么在STM32中使用C语言中大名鼎鼎的printf函数。printf函数属于C语言自带的函数，需要在keil中勾选**Use MicroLIB**选项*（怎么勾选？百度一下，你就知道）*。然后在代码中的任何一个位置加入下面一段代码：
```c
#pragma import(__use_no_semihosting)
//标准库需要的支持函数
struct __FILE
{
	int handle;
};
FILE __stdout;
//定义_sys_exit()以避免使用半主机模式
void _sys_exit(int x)
{
	x = x;
}
//重定义 fputc 函数
int fputc(int ch, FILE *f)
{
	while((USART1->SR&0X40)==0);//循环发送,直到发送完毕
	USART1->DR = (u8) ch;
	return ch;
}
```
上面的fputc函数是被printf函数调用的函数之一，只需要重定义它即可实现将printf的内容通过串口发送出去。
#TIM
TIM分为**高级定时器**、**通用定时器**和**基本定时器**，其中基本定时器*（一般为TIM6和TIM7）*的功能最简单，只有定时的功能，一般用作时钟基源*（比如**FreeRTOS操作系统**的时钟基源）*；通用定时器在基本的定时功能的基础上多出了**输出比较**和**输入捕获**功能，输出比较可以输出周期性的方波*（比如**PWM**波和**PPM**波）*，输入捕获可以读取输入信号的高电平和低电平的时间，进而可以计算出信号的周期和占空比，这两者都应用十分广泛；高级定时器除了上述功能之外，还有还包含一些与电机控制和数字电源应用相关的功能，比方带死区控制的互补信号输出、紧急刹车关断输入控制等，这些功能可以用于控制高级的工业应用当中。
在我们的日常开发中只需要掌握通用定时器的定时、输出比较、和输入捕获功能就足够了。
##定时功能
TIM最简单的功能就是定时功能，它对应的有一个中断函数，可以实现定时执行某个操作。TIM定时器要使用之前需要初始化，主要注意两个寄存器的值，一个是**预分频**寄存器的值，一个是**ARR**寄存器的值，这两个寄存器决定了定时器的计数周期，也就是定时器中断发生的频率。
这里要注意的是TIM定时器的定时属于硬件定时，是不允许超时的*（事实上不论是什么定时器都不建议超时）*，如果超时就会卡死程序。TIM定时器使用前要打开定时器，函数如下：
```c
HAL_TIM_Base_Start_IT(TIM_HandleTypeDef * htim);  //定时器中断使能函数
```
对应的定时器中断回调函数为：
```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef * htim)
{  
  //用户自己的中断函数逻辑
  //your code here ...
}
```
##PWM
定时器的一个重要应用就是产生**PWM波**。PWM波的应用非常广泛，无论是舵机，还是蜂鸣器，还是航模电调，都是使用PWM驱动。PWM波有两个参数，一个是周期，一个是占空比，它们分别对应TIM的ARR寄存器和CCRx*（x是定时器的通道号）*寄存器。如果再程序中途想改变PWM波的占空比，需要直接操作寄存器，因为HAL库并没有相关函数。
```c
TIM1->CHANNEL1 = 1000;			//将TIM1的通道1对应的CCR1寄存器赋值为1000
```
这里可以给大家介绍一下**影子寄存器**的概念。我们用户所操作的所有寄存器，都不是真正起作用的寄存器，真正起作用的寄存器是影子寄存器。我们所操作的寄存器叫**预装载寄存器**，影子寄存器的值会随着预装载寄存器的改变而改变，但**不是立即改变**，其修改值只能通过更新事件才能生效。因为在定时器计数的过程中影子寄存器的值直接改变可能会引发错误。
使用定时器的PWM波功能也需要一个开启定时器的操作，函数如下：
```c
HAL_TIM_PWM_Start(TIM_HandleTypeDef * htim, uint32_t Channel);  //PWM波产生使能函数
```
PWM波一般不会用到中断函数。
##PWM_Read
我们一般都是产生PWM波去控制外设，但大家可以换个角度想一想，如果让我们自己来开发一个舵机呢，我们就需要读取输入的PWM波的占空比和周期，最笨的方法就是在死循环里面不断地扫面IO的电平，当电平变化的时候开始利用延时函数计时，这种方法效率不高且精度很低。另一种方法就是使用定时器的输入捕获功能。
定时器的输入捕获有三种模式：**上升沿捕获**、**下降沿捕获**和**上升下降沿捕获** *（有没有感觉非常像GPIO外部中断的触发方式？）*，这三种状态所对应的中断触发条件不一样。显然，如果我们要检测PWM波的高电平，则初始要设置为上升沿触发，触发之后改为下降沿触发。
那么怎么计时呢？定时器内部有一个不断在计数的寄存器CNT，计数方式又大体上分为**向上计数**、**向下计数**和**中间计数**。拿向上计数举例，CNT从0按预分频后的频率计数到ARR的值之后自动重载为0，然后继续计数。在输入捕获模式下，CNT的值会被赋值给CCRx，所以我们只需要在每次触发中断之后**将CNT清零**，然后在下一次中断里面获取CCRx的值，再结合定时器的频率就可以计算出对应的时间。
同上，使用输入捕获功能需要一个开启操作：
```c
//在软件里面已经设置好了为上升沿触发模式
HAL_TIM_IC_Start_IT(&htim4, TIM_CHANNEL_1);    //这里是打开了TIM4通道1的输入捕获功能
```
相对应的输入捕获的中断回调函数为：
```c
void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
	//用户自己的中断函数逻辑
	//your code here...
}
```
这里给出一份读取PWM波高电平和低电平时间的代码：
```c
/*
 * 使用的是TIM4的通道1作为输入捕获口
 * 预分频后定时器频率为1MHz，即计数周期为1微秒1次
 */

uint32_t pwm_high_level_time;    //PWM波高电平的时间
uint32_t pwm_low_level_time;     //PWM波低电平的时间
int tim_mode_raise_or_falling = 0;//0代表上升，1代表下降

void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
  if(tim_mode_raise_or_falling == 0)			//如果是上升沿触发
  {
    pwm_low_level_time = HAL_TIM_ReadCapturedValue(&htim4,TIM_CHANNEL_1) + 1;		//记录低电平的时间（第一次触发该数值无效）
    __HAL_TIM_SET_COUNTER(&htim4,0);		//清零定时器计数CNT
    TIM_RESET_CAPTUREPOLARITY(&htim4, TIM_CHANNEL_1);		//重置定时器配置
    TIM_SET_CAPTUREPOLARITY(&htim4,TIM_CHANNEL_1,TIM_ICPOLARITY_FALLING);		//配置定时器为下降沿触发模式
    tim_mode_raise_or_falling = 1;		//中断模式标志位改变
  }
  else if(tim_mode_raise_or_falling == 1)			//如果是下降沿触发
  {
    pwm_high_level_time = HAL_TIM_ReadCapturedValue(&htim4,TIM_CHANNEL_1) + 1;		//记录高电平的时间
    __HAL_TIM_SET_COUNTER(&htim4,0);		//清零定时器计数CNT
    TIM_RESET_CAPTUREPOLARITY(&htim4, TIM_CHANNEL_1);		//重置定时器配置
    TIM_SET_CAPTUREPOLARITY(&htim4,TIM_CHANNEL_1,TIM_ICPOLARITY_RISING);		//配置定时器为上升沿触发模式
    tim_mode_raise_or_falling = 0;		//中断模式标志位改变
  }
}
```
细心的同学可能会发现，在记录时间的那条语句的最后有一个`+1`的操作，这里我引用一段我看到过的描述来解释这一现象（原文见[https://wenku.baidu.com/view/dcd8f0f67f1922791688e8f6.html](https://wenku.baidu.com/view/dcd8f0f67f1922791688e8f6.html) ）：
> PWM模式：
> PWM边沿对齐PWM1模式，向上计数时，CCRx正确取值范围为（0~ARR）：
> CCRx = 0时，产生全无效电平（产生占空比为0%的PWM波形）
> CCRx <= ARR时，产生CCRx个有效电平(产生占空比为 CCRx/(ARR+1)*100% 的PWM波形)。
> CCRx > ARR时，产生全有效电平。
> PWM边沿对齐PWM1模式，向下计数时，CCRx正确取值范围为(0~ARR)：
> CCRx = 0时，不能产生占空比 0% 的PWM波形(产生占空比为1/(ARR+1)*100%的PWM波形)。
> CCRx <= ARR时，产生CCRx+1个有效电平(产生占空比为 (CCRx+1)/(ARR+1)*100% 的PWM波形)。
> CCRx > ARR时，产生全有效电平。
> 
> 捕获脉冲：
> 自动复位计数器方式下的PWM输入信号测量
> 在该模式下，可以方便地测试输入信号的周期(频率/转速)和占空比。
> TIMx_CCR1的 寄存器值+1 就是周期计数值，TIMx_CCR2的 寄存器值+1 就是高电平计数值。
> 占空比=(TIMx_CCR2+1)/(TIMx_CCR1+1)*100%

从上面也可以看出，我们配置定时器的ARR的值的时候要减一个1，比方说我们想要周期为20000，那么ARR的值应该赋值为19999，因为CNT的数值是从0~19999，一共20000次计数。
#ADC
STM32F1 系列芯片共两个**ADC**模块，每个**ADC**模块有9个通道，共18个通道。**ADC**的工作模式有**单次模式**、**连续模式**、**扫描模式**和**间断模式**。本次培训主要讲解了连续模式的使用，并在**DMA**模式下读取数据。

ADC使用DMA时要注意选择**Circular**模式。另外ADC的读取到的有效数字是后十二位，对应的电压范围是0~3.3V，在中断函数中需要计算。涉及到的函数如下:
```c
HAL_ADC_Start_DMA(ADC_HandleTypeDef * hadc, uint32_t * pData, uint32_t Length);  //开启使能ADC功能
```
ADC的中断回调函数如下：
```c
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef * hadc)
{  
  your code here ...
  //用户自己的中断函数逻辑
}
```
#IIC
STM32F1 系列芯片共两个**IIC**模块，IIC开发最重要的东西就是**时序图**，只要看懂了时序图一切都显得极为简单。

IIC 的使用分为**使用IO口模拟时序**和**直接使用硬件寄存器**两种方式实现IIC通信。本次培训主要讲解了使用STM32硬件自带的IIC模块的实现方式。主要涉及以下函数:
```c
HAL_I2C_Mem_Write(I2C_HandleTypeDef * hi2c, uint16_t DevAddress, uint16_t MemAddress, uint16_t MemAddSize, uint8_t * pData, uint16_t Size, uint32_t Timeout);  //将数据写入指定地址的从机
```
IIC没有涉及到中断函数。
#WWDG
STM32 分为**独立看门狗**和**窗口看门狗**，其中独立看门狗时钟源独立，用于监控硬件异常，窗口看门狗时钟源不独立，用于监控软件异常。

窗口看门狗（ Window Watch Dog Timer ）初始化之后需要喂狗，喂狗有**时间上限和下限**。涉及到的函数如下：
```c
HAL_WWDG_Refresh(WWDG_HandleTypeDef * hwwdg);  //重载窗口看门狗递减计数器（喂狗）
```
本次培训没有讲解WWDG的中断。
