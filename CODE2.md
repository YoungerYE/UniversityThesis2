# UniversityThesis2
本科课程设计——基于单片机的停车场车辆统计系统
#include <reg52.h>	         //调用单片机头文件
#define uchar unsigned char  //无符号字符型 宏定义	变量范围0~255
#define uint  unsigned int	 //无符号整型 宏定义	变量范围0~65535

sbit beep = P1^4; //蜂鸣器IO口定义 

sbit hw_jin = P2^1;   //红外传感器IO口定义 
sbit hw_chu = P2^0;   //红外传感器IO口定义 
uchar menu_1;	      //设置参数用

uint num;  //计的数
uint jin,chu;

uchar set_num = 30; // 设置总的车位 	

sbit red   = P1^3;	   //红色发光二极管定义
sbit green = P1^5;	   //绿色发光二极管定义

void delay_1ms(uint q)
{
	uint i,j;
	for(i=0;i<q;i++)
		for(j=0;j<120;j++);
}

sbit rs=P1^0;	 //寄存器选择信号 H:数据寄存器  	L:指令寄存器
sbit rw=P1^1;	 //寄存器选择信号 H:数据寄存器  	L:指令寄存器
sbit e =P1^2;	 //片选信号   下降沿触发

* 名称 : delay_uint()
* 功能 : 小延时。
* 输入 : 无
* 输出 : 无

void delay_uint(uint q)
{
	while(q--);
}


* 名称 : write_com(uchar com)
* 功能 : 1602命令函数
* 输入 : 输入的命令值
* 输出 : 无

void write_com(uchar com)
{
	e=0;
	rs=0;
	rw=0;
	P0=com;
	delay_uint(25);
	e=1;
	delay_uint(100);
	e=0;
}

* 名称 : write_data(uchar dat)
* 功能 : 1602写数据函数
* 输入 : 需要写入1602的数据
* 输出 : 无

void write_data(uchar dat)
{
	e=0;
	rs=1;
	rw=0;
	P0=dat;
	delay_uint(25);
	e=1;
	delay_uint(100);
	e=0;	
}

void write_sfm4(uchar hang,uchar add,uint date)
{
	if(hang==1)   
		write_com(0x80+add);
	else
		write_com(0x80+0x40+add);
	write_data(0x30+date/1000%10);
	write_data(0x30+date/100%10);
	write_data(0x30+date/10%10);
	write_data(0x30+date%10);	
}

void write_string(uchar hang,uchar add,uchar *p)
{
	if(hang == 1)   
		write_com(0x80+add);
	else
		write_com(0x80+0x40+add);
	while(1)														 
	{
		if(*p == '\0')  break;
		write_data(*p);
		p++;
	}	
}

void init_1602()	//lcd1602初始化
{
	write_com(0x38);	
	write_com(0x0c);
	write_com(0x06);
	delay_uint(1000);
	write_string(1,0,"   SY:0000       ");	
	write_string(2,0," J:0000  C:0000   ");	
	write_sfm4(2,2,jin);  //显示人数
	write_sfm4(1,8,num);  //显示人数
	write_sfm4(2,12,chu);  //显示人数
}

uchar key_can;	 //按键值

void key()	     //独立按键程序
{
	static uchar key_new;
	key_can = 20;               //按键值还原
	P3 |= 0xf0;
	if((P3 & 0xf0) != 0xf0)		//按键按下
	{
		delay_1ms(1);	     	//按键消抖动
		if(((P3 & 0xf0) != 0xf0) && (key_new == 1))
		{						//确认是按键按下
			key_new = 0;
			switch(P3 & 0xf0)
			{
				case 0xd0: key_can = 3; break;	   //得到k1键值
				case 0xb0: key_can = 2; break;	   //得到K2键值
				case 0x70: key_can = 1; break;	   //得到k3键值
			}
		}			
	}
	else 
		key_new = 1;	
}

void key_with()
{
	if(key_can == 1)	//设置键
	{
		menu_1 ++;
		if(menu_1 >= 2)
		{
			menu_1 = 0;
			init_1602();  //lcd1602初始化	
		}
		if(menu_1 == 1)				 //初始化显示
		{
			write_string(1,0,"     SET Z       ");
			write_string(2,0,"                 ");
		}
	}
	if(menu_1 == 0)   			//倒计时器按键操作开始 暂停
	{ 		
		if(key_can == 2)  //清零
		{
			num = 0;
			jin = 0;
			chu = 0;
			num  = set_num - jin + chu;	  //计算剩余车位
			write_sfm4(2,2,jin);  //显示人数
			write_sfm4(1,8,num);  //显示人数
			write_sfm4(2,12,chu);  //显示人数
		}
	}
	if(menu_1 == 1)				//设置倒计时器开始数
	{
		if(key_can == 2)
		{
			set_num ++ ;		// 设置数加
			if(set_num > 9999)
				set_num = 9999;	//最大加到99	
		}
		if(key_can == 3)
		{
			set_num -- ;		// 设置数减
			if(set_num <= 1)
				set_num = 1;	//最大减到1
		}
		write_sfm4(2,5,set_num);  //显示人数
		write_com(0x80+0x40+7);             //将光标移动到秒个位
		write_com(0x0f);                    //显示光标并且闪烁	
	}
	beep = 0;	  //打开蜂鸣器
	delay_1ms(50);
	beep = 1;	  //关闭蜂鸣器
	
}  
void hw_jin_dis()	//红外计数
{
	if(hw_jin == 0)		//计数
	{
		delay_1ms(1);	     	//消抖动
		if(hw_jin == 0)
		{						//确认
			jin ++;
			if(jin >= 9999)
				jin = 9999;
			num  = set_num - jin + chu;	  //计算剩余车位
			write_sfm4(2,3,jin);  //显示人数
			write_sfm4(1,7,num);  //显示人数
			if(num == 0)		  //为0时报警
			{
				beep = 0;
				delay_1ms(200);
				beep = 1;	 					
				delay_1ms(200);
				beep = 0;
				delay_1ms(200);
				beep = 1;	 					
				delay_1ms(200);
				beep = 0;
				delay_1ms(200);
				beep = 1;	 					
			}
		}			
	}
}

void hw_chu_dis()	//红外计数
{
	if(hw_chu == 0)		//计数
	{
		delay_1ms(1);	     	//消抖动
		if(hw_chu == 0)
		{						//确认
			chu ++;
			if(chu >= 9999)
				chu = 9999;
			num  = set_num - jin + chu;	  //计算剩余车位
			write_sfm4(2,11,chu);  //显示人数
			write_sfm4(1,7,num);  //显示人数
		}
	}
}

uchar nnum;
   
void main()
{
	init_1602();	//lcd1602初始化
	while(1)
	{
		key();			   //按键扫描函数
		if(key_can < 10)
			key_with();    //按键执行函数
		hw_jin_dis();	//红外计数	
		hw_chu_dis();	//红外计数	
		if(num == 0)
		{
			red  = 0;  green = 1; //车位为0 红灯亮
		}else 
		{
			red  = 1;  green = 0; //绿灯亮
		}						
		delay_1ms(100);
	}
}
