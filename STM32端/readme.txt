**********
这是关于STM32F407IGHX与ROS2进行虚拟串口通信的具体代码
**********
简介：该内容利用USB-CDC来创建一个虚拟串口，然后利用结构体等结构来规定了数据的具体形式，从而便于各种函数的封装。
**********
注意：该工程比较复杂，有些内容是直接在初始化好的工程上修改得来而并不是通过STM32CubeIDE来初始化的，所以不要轻易再次初始化，
否则很有可能丢失位于USB_DEVICE中App的usbd_cdc_if.c和usbd_cdc_if.h中修改过的发送/接收函数，从而产生不必要的报错。
**********
基本介绍（为免麻烦，我先不写函数的具体参数了，只写函数名称和括号）：

注意事项：


1、usbd_cdc_if.c
里面包含了热心网友重新编写的
CDC_Init_FS() , CDC_DeInit_FS() , CDC_Control_FS() , CDC_Receive_FS() , CDC_Transmit_FS() , CDC_TransmitCplt_FS()五个函数。
我暂时只写下我了解的部分
CDC_Receive_FS()是接收上位机数据的函数，你可以填入一个数组指针以及需要接收的数据长度(但是为了内部函数的正常工作，所以你要引用传入的长度的地址，如&Len)。
CDC_Transmit_FS()是向上位机发送数据的函数，填入一个数组指针以及需要接收的数据长度。
CDC_TransmitCplt_FS()是中断服务函数，可以让STM32中断来接收数据。

2、usbd_cdc_if.h
里面仅包含上述5个函数的定义。

3、Serial.c
里面包含CDC_SendFeed() ，Pack_And_Send_Data_ROS2()，UnPack_Data_ROS2()，CDC_Receive_ROS2()，Get_CRC16_Check_Sum()等函数的具体实现
CDC_SendFeed() 发送反馈数据的函数。
Pack_And_Send_Data_ROS2() 打包和发送反馈数据的函数。
UnPack_Data_ROS2() 解包函数
CDC_Receive_ROS2()  接收函数
Get_CRC16_Check_Sum()CRC校验码生成，该校验码是以字节方式生成的、

5、Serial.h
里面包含了反馈数据，控制数据的结构体的具体定义。
注意结构体的内容，如typedef struct ReceivePacket _receive_packet;  的 _receive_packet，这是结构体的别名
而}__attribute__((packed))_receivepacket,*_receive_packetinfo;  这个 _receive_packetinfo 就是结构体的指针。
这种别名以及指针的写法是被C所允许的。

6、usbd_cdc_if.c
里面虽然不是我写的，但是包含了USB-CDC的设备描述符的定义以及发送，有可能会被用到。

**********
基本操作：
发送数据：
先定义对应结构体以及相应合适的缓冲数组，然后再给结构体分配空间，赋值（可以不赋，等待读入数值），然后使用CRC校验码生成程序生成校验码。
接着利用打包和发送函数Pack_And_Send_Data_ROS2()，打包好数据，并发送数据。
接收数据
先定义对应结构体以及相应合适的缓冲数组，然后再给结构体分配空间，赋值（可以不赋，等待读入数值）。
然后利用接收函数CDC_Receive_ROS2()（接收函数内部写好了解包函数），获得含有数据的结构体。
举个例子：
		_send_packetinfo sd;
		uint8_t RecePackage[sizeof(_receive_packet)];
		sd = (_sendpacket *)malloc(sizeof(_sendpacket));
		receinfo = (_receive_packetinfo)malloc(sizeof(_receive_packet));//为结构体指针赋予空间
		sd->header = 0X5A;
		sd->robot_color = 1;
		sd->task_mode= 2;
		sd->reserve = 5;

		sd->checksum=0X00;
		memset(receinfo,0,sizeof(_receivepacket));
	  /* Infinite loop */
	  for(;;)
	  {
		  /***向上位机发送数据***/
		  sd->pitch = angle_pitch;
		  sd->yaw = angle_yaw;
		  if(!receinfo->tracking)
		  {
			  sd->aim_x = 0;
			  sd->aim_y = 0;
			  sd->aim_z = 0;
		  }
		  else{
			  sd->aim_x = aim[0];
			  sd->aim_y = aim[1];
			  sd->aim_z = aim[2];
		  }
		  //memmove(temp_CRC,sd,10);
//		  sd->checksum=Get_CRC16_Check_Sum(sd, temp_CRC, SendData, 24, 0xFFFF);
		  Pack_And_Send_Data_ROS2(sd,(size_t)sizeof(_sendpacket));
		  CDC_Receive_ROS2(RecePackage, (size_t)sizeof(_receive_packet), receinfo);
**********

参考文档：
https://blog.csdn.net/sinat_16643223/article/details/108894564     ros中使用serial包实现串口通信
https://blog.csdn.net/qq_33904382/article/details/112718948  怎样用串口发送结构体-简单协议的封包和解包
https://blog.csdn.net/Naisu_kun/article/details/118192032   STM32 USB使用记录：使用CDC类虚拟串口（VCP）进行通讯
https://www.cnblogs.com/MORAN-123/p/17084387.html  关于STM32CubeIDE无法正常启动GDB服务端的解决办法

**********
备注：
如果你无法获取设备描述符，从而无法连接的话，可能是你的时钟配错了，建议参考一下该工程的时钟配置。

**********
欢迎参考
**********








