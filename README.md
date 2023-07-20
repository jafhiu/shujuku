# shujuku
源代码库
/*
 * user_app_ui.c
 *
 * created: 2022/7/13
 *  author:
 */
/**
  * @file   menutest.h
  * @author  guoweilkd
  * @version V0.0.0
  * @date    2018/06/30
  * @brief  菜单使用Demo
  */
#include "user_app_ui.h"
#include "keyDriverPort.h"
#include "lkdGui.h"
#include "user_rtc_drv.h"
#include "ls1x_rtc.h"
#include "ls1b_gpio.h"





#include "ls1x_fb.h"
#include "user_gauge_drv.h"



#include "ls1x_i2c_bus.h"

#include "i2c/ads1015.h"
/* 菜单按键 */
enum MenuKeyIs keyStatus;
/* 菜单句柄 */
lkdMenu HmiMenu;
/* 菜单栈 */
#define MENUSTACK_NUM 8
MenuStack userMenuStack[MENUSTACK_NUM];

/* 函数申明 */
static void HomeFun(void *param);
static void MenuFun(void *param);


/* 窗口定义 */
lkdWin homeWin = {5,5,470,790,"水质监测系统",HomeFun,NULL};/* 桌面窗口 */
lkdWin menuWin = {5,5,470,790,"水质监测系统",MenuFun,NULL};/* 菜单窗口 */





//lkdMenuNode Node2 = {2,"水体温度",		NULL,	NULL,       NULL};
//lkdMenuNode Node1 = {1,"水体污浊程度",		&Node2,	NULL,	NULL	};
lkdMenuNode Node0 = {0,"root",		NULL,	NULL,		NULL};


/**
  *@brief  菜单项处理
  *@param  step 步骤 pNode 菜单节点
  *@retval None
  */
static void MenuItemDealWith(lkdMenuNode *pNode)
{
    if(pNode->pSon != NULL) //展开选中节点的菜单
    {
        GuiMenuCurrentNodeSonUnfold(&HmiMenu);
    }
    else if(pNode->user != NULL) //打开菜单对应的窗口
    {
        GuiWinAdd(pNode->user);
    }
}

/**
  *@brief  菜单控制函数
  *@param  None
  *@retval None
  */
void MenuControlFun(void)
{
    switch(keyStatus)
    {
        case MKEY_UP:
            GuiMenuItemUpMove(&HmiMenu);
            break;
        case MKEY_DOWN:
            GuiMenuItemDownMove(&HmiMenu);
            break;
        case MKEY_LEFT:
        case MKEY_CANCEL:
            GuiMenuCurrentNodeHide(&HmiMenu);
            if(HmiMenu.count == 0) //检测到菜单退出信号
            {
                GuiWinDeleteTop();
            }
            break;
        case MKEY_RIGHT:
        case MKEY_OK:
            MenuItemDealWith(GuiMenuGetCurrentpNode(&HmiMenu));
            break;
    }
}

/**
  *@brief  菜单初始化
  *@param  None
  *@retval None
  */
void UserMenuInit(void)
{
    HmiMenu.x = 40;
    HmiMenu.y = 55;
    HmiMenu.wide = 440;
    HmiMenu.hight = 760;
    HmiMenu.Itemshigh = 40;
    HmiMenu.ItemsWide = 400;
    HmiMenu.stack = userMenuStack;
    HmiMenu.stackNum = 8;
    HmiMenu.Root = &Node0;/* 菜单句柄跟菜单节点绑定到一起 */
    HmiMenu.MenuDealWithFun = MenuControlFun;/* 菜单控制回掉函数 */
    GuiMenuInit(&HmiMenu);
}



#define RES2 820.0   //运放电阻，与硬件电路有关
#define ECREF 200.0 //极片输入电压，与硬件电路相关
/**
  *@brief  桌面窗口实体
  *@param  None
  *@retval None
  */
  
  float EC_voltage;
float EC_value=0.0;
float temp_data=250;
float compensationCoefficient1=1.0;//温度校准系数
float kValue1=1.0;
float kValue_Low=898.7;  //校准时进行修改
float kValue_High=958.465; //校准时进行修改
float rawEC=0.0;
float EC_valueTemp=0.0;


float TDS=0.0,TDS_voltage;
float TDS_value=0.0,voltage_value;
float compensationCoefficient=1.0;//温度校准系数
float compensationVolatge;
float kValue=0.9;
float TEMP_Value=0.0;
// 用于保存转换计算后的电压值
float ADC_ConvertedValueLocal;
  
short tempp=0;
uint8_t time_Flag=0;
char tbuf1[50]={0},tbuf2[50]={0},tbuf3[50]={0},tbuf4[50]={0},sbuf1[50]={0},sbuf2[50]={0},sbuf3[50]={0},sbuf4[50]={0},rt;
unsigned short  adc1=0, adc2=0, adc3=0, adc4=0;
float out_v,in_v1,in_v2,in_v3,in_v4;
float turbidity_=0,PH_Value=0,EC_Value_=0,TDS_Value_=0;
char  turbidity_Flag=0;
unsigned int PH_Value_t=0;
 

/**************************************************************
*功  能：电导率信号采集
*参  数: 无
*返回值:
无
**************************************************************/
float EC_Test(float Get_Adc)
{

	EC_valueTemp=hwd_ds18b20_get_temp();

	EC_voltage=(float)Get_Adc;
	rawEC = 1000*EC_voltage/RES2/ECREF;
	EC_valueTemp=rawEC*kValue1;
	if(EC_valueTemp>2.0)
	{
		kValue1=kValue_High;
	}
	else if(EC_valueTemp<=2.0)
	{
		kValue1=kValue_Low;
	}
	EC_value=rawEC*kValue1;
	compensationCoefficient1=1.0+0.0185*((temp_data/10)-25.0);
	EC_value=EC_value/compensationCoefficient1;
    return EC_value;
}




/**************************************************************
*功  能：TDS信号采集
*参  数: 无
*返回值:
无
**************************************************************/
float Tds_Test(float Get_Adc)
{

	TEMP_Value=hwd_ds18b20_get_temp();

	ADC_ConvertedValueLocal=Get_Adc;
	compensationCoefficient=1.0+0.02*((TEMP_Value/10)-25.0);
	compensationVolatge=ADC_ConvertedValueLocal/compensationCoefficient;
	TDS_value=(133.42*compensationVolatge*compensationVolatge*compensationVolatge - 255.86*compensationVolatge*compensationVolatge + 857.39*compensationVolatge)*0.5*kValue;


	//if((TDS_value>1400))

	//{
		//TDS_value=1400;
	//}
	//if(TDS_value<=0)
	//{

		//TDS_value=00;
	//}
    return TDS_value;


}

	
static void HomeFun(void *param)
{
    static str[50]= {0};
    adc1 = get_ads1015_adc(busI2C0, ADS1015_REG_CONFIG_MUX_SINGLE_0);

    adc2 = get_ads1015_adc(busI2C0, ADS1015_REG_CONFIG_MUX_SINGLE_1);

    adc3 = get_ads1015_adc(busI2C0, ADS1015_REG_CONFIG_MUX_SINGLE_2);

    adc4 = get_ads1015_adc(busI2C0, ADS1015_REG_CONFIG_MUX_SINGLE_3);

    
    
    in_v1=6.144*2*adc1/4096;
    in_v2=6.144*2*adc2/4096;
    in_v3=6.144*2*adc3/4096;
    in_v4=6.144*2*adc3/4096;
    delay_ms(500);
    //水体PH值
    
    PH_Value_t+=in_v1;

	if(time_Flag=200)
    {
        //PH_Value=PH_Value_t/100+((PH_Value_t%100)*0.01);
        //PH_Value=PH_Value_t/100+((PH_Value_t%100)*0.01);
        
        PH_Value=-5.8887*in_v1+21.677;
        	if(PH_Value<=0.0)
	       {
		      //PH_Value=0.0;
		      PH_Value=PH_Value_t/100+((PH_Value_t%100)*0.01);
	       }
	       if(PH_Value>=14.0)
	       {
		      //PH_Value=14.0;
		      PH_Value=PH_Value_t/100+((PH_Value_t%100)*0.01);
	       }
	    sprintf(str,"  水质PH值：%.02f   ",(float)PH_Value);
        GuiRowText(0,180,460, FONT_MID,str);
        time_Flag=0;
        PH_Value_t=0;
        
    }

    time_Flag++;


    //水体浊度
    turbidity_ = in_v2*100/3.3;
    if(turbidity_ > 100) turbidity_ = 100;
    if(in_v2>2.96)
        turbidity_Flag=1;
    else if(in_v2>2.64)
        turbidity_Flag=2;
    else if(in_v2>1.84)
        turbidity_Flag=3;
    else
        turbidity_Flag=4;

    sprintf(str,"  水体污浊程度:%d   ",(int)turbidity_);
    GuiRowText(0, 250,460, FONT_MID,str);
        sprintf(str,"  水体污浊等级：%d 级   ",turbidity_Flag);
    GuiRowText(0, 290,460, FONT_MID,str);
    
    EC_Value_=EC_Test(in_v3);
    sprintf(str,"  水体电导率:%.02f ",(float)EC_Value_);
    GuiRowText(0, 350,460, FONT_MID,str);

    //printk("通道一%f\n",in_v1);
    //printk("通道二%f\n",in_v2);
    //printk("通道三%f\n",in_v3);
    //printk("通道四%f\n\n",in_v4);
    TDS_Value_=Tds_Test(in_v4);
    
    sprintf(str,"  固体溶解度:%.02f  ",(float)TDS_Value_);
    GuiRowText(0,430,460, FONT_MID,str);
    
    //温度
    out_v=(hwd_ds18b20_get_temp()/10)+((hwd_ds18b20_get_temp()%10)*0.1);
    
    sprintf(str,"  温度:%.02f ",(float)out_v);
    GuiRowText(0, 530,460, FONT_MID,str);
    delay_ms(10);

     switch(keyStatus)
    {

        case MKEY_UP:

            break;
        case MKEY_DOWN:

            break;
        case MKEY_LEFT:

            break;
        case MKEY_RIGHT:
            break;
        case MKEY_CANCEL:
            GuiWinDeleteTop();
            GuiMenuRedrawMenu(&HmiMenu);
            break;
        case MKEY_OK:

            break;
    }

}


/**
  *@brief  菜单窗口实体
  *@param  None
  *@retval None
  */
static void MenuFun(void *param)
{
    if(HmiMenu.count == 0)
    {
        GuiMenuCurrentNodeSonUnfold(&HmiMenu);
    }
    HmiMenu.MenuDealWithFun();
}


/**
  *@brief  用户菜单主函数
  *@param  None
  *@retval None
  */
void userAppMain(void)
{
    keyStatus = GetMenuKeyState();
    SetMenuKeyIsNoKey();
}

/**
  *@brief  用户应用初始化
  *@param  None
  *@retval None
  */
void userAppInit(void)
{
    UserMenuInit();
    GuiSetForecolor(1);
    GuiSetbackcolor(0);
    GuiWinAdd(&homeWin);
}

/* END */
