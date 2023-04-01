---
title: ZigBee协议栈探究---（二）数据的接收与发送
date: 2021-03-12 10:03:41
tags: [ZigBee,CC2530,协议栈]
---

#  ZigBee协议栈探究---（二）数据的接收与发送

## 1.回顾

上一章讲解了ZigBee协议栈中函数的运行走向，了解了ZigBee的任务最终在`ProgressEvent`这个函数中执行，那么在协议栈中这个函数在不改动的情况下究竟执行了什么？我们又应该如何改动它呢？我们又如何进行数据的发送与接收呢？

<!--more-->

## 2.从ProcessEvent函数开始讨论数据接收处理

首先来看看`Smart_home_ProcessEvent()`函数究竟做了什么

```c
/*********************************************************************
 * @fn      Smart_home_ProcessEvent
 *
 * @brief   Generic Application Task event processor.
 *
 * @param   task_id  - The OSAL assigned task ID.
 * @param   events   - Bit map of events to process.
 *
 * @return  Event flags of all unprocessed events.
 */
UINT16 Smart_home_ProcessEvent( uint8 task_id, UINT16 events )
{
  (void)task_id;  // Intentionally unreferenced parameter
  
  if ( events & SYS_EVENT_MSG )
  {
    afIncomingMSGPacket_t *MSGpkt;

    while ( (MSGpkt = (afIncomingMSGPacket_t *)osal_msg_receive( Smart_home_TaskID )) )
    {
      switch ( MSGpkt->hdr.event )
      {
      case ZDO_CB_MSG:
        Smart_home_ProcessZDOMsgs( (zdoIncomingMsg_t *)MSGpkt );
        break;
          
      case KEY_CHANGE:
        Smart_home_HandleKeys( ((keyChange_t *)MSGpkt)->state, ((keyChange_t *)MSGpkt)->keys );
        break;

      case AF_INCOMING_MSG_CMD:
        Smart_home_ProcessMSGCmd( MSGpkt );
        break;

      default:
        break;
      }

      osal_msg_deallocate( (uint8 *)MSGpkt );
    }

    return ( events ^ SYS_EVENT_MSG );
  }

  if ( events & Smart_home_SEND_EVT )
  {
    Smart_home_Send();
    return ( events ^ Smart_home_SEND_EVT );
  }

  if ( events & Smart_home_RESP_EVT )
  {
    Smart_home_Resp();
    return ( events ^ Smart_home_RESP_EVT );
  }

  return ( 0 );  // Discard unknown events.
}

```

在这个函数中最引人瞩目的应该就是`MSGpkt`以及一大堆的宏调用了。让我们来逐一分析一下。首先就是`MSGpkt`，很明显这个函数的类型是`afIncomingMSGPacket_t`的指针，我们来看一下这个结构体的构造

```c
typedef struct
{
  osal_event_hdr_t hdr;     /* OSAL信息头部 */
  uint16 groupId;           /*组号，未设置时默认为0 */
  uint16 clusterId;         /* 信息簇号 */
  afAddrType_t srcAddr;     /* 源地址，如果端点号为STUBAPS_INTER_PAN_EP,
                               就是InterPAN 信息 */
  uint16 macDestAddr;       /* MAC头 目的短地址 */
  uint8 endPoint;           /* 目的端点号 */
  uint8 wasBroadcast;       /* 如果是广播则为真 */
  uint8 LinkQuality;        /* 收到数据帧的链接质量 */
  uint8 correlation;        /* 接收到的数据帧的原始相关值 */
  int8  rssi;               /* 接收的射频功率，单位dBm */
  uint8 SecurityUse;        /* 不推荐使用 */
  uint32 timestamp;         /*  MAC收据时间戳 */
  uint8 nwkSeqNum;          /* 网络报头帧序列号 */
  afMSGCommandFormat_t cmd; /* 应用层数据 */
} afIncomingMSGPacket_t;
```

通过阅读注释，很明显能明白这是数据帧报文段的定义。也就是说`MSGpkt`是一个初始化的报文段指针。那么这个报文段是由谁给出来的呢？再来看下面这一句

```c
 while ( (MSGpkt = (afIncomingMSGPacket_t *)osal_msg_receive( Smart_home_TaskID )) 
```

对于`osal_msg_receive( Smart_home_TaskID )`这一部分，这里很容易从函数名看出指的是接收到的数据。这里暂且不深入探究，我更好奇的是：在while中按case进行划分时，这些case都是什么？又都可以设置为什么呢？我们随便挑几个来看一看。

可以看出，指引case的条件是`MSGpkt->hdr.event`，在上面可以找到，hdr的类型是`osal_event_hdr_t`,这个数据结构定义是这样的

```c
typedef struct
{
  uint8  event;
  uint8  status;
} osal_event_hdr_t;
```

也就是说，这个结构体中只有uint8 的event和status两个。再回头来看这些case。通过观察我们发现，这几个case并不处于同一个.h文件，而是分散到几个.h文件中。但都具有一个比较共同的标识 ：`Global Generic System Message`，在下面我尽我所能的整理了一下可能出现的case，以供参考

```c
/*comdef.h*/
#define KEY_CHANGE                0xC0    // 按键事件
/*ZComDef.h*/
#define SPI_INCOMING_ZTOOL_PORT   0x21    // 来自ZTool端口的原始数据（未实现）
#define SPI_INCOMING_ZAPP_DATA    0x22    // 来自ZAPP端口的原始数据（参见serialApp.c）
#define MT_SYS_APP_MSG            0x23    // 来自MT Sys消息的原始数据
#define MT_SYS_APP_RSP_MSG        0x24    // MT Sys消息的原始数据输出
#define MT_SYS_OTA_MSG            0x25    // MT OTA Rsp的原始数据输出

#define AF_DATA_CONFIRM_CMD       0xFD    // 数据确认
#define AF_INCOMING_MSG_CMD       0x1A    // 收到的MSG类型消息
#define AF_INCOMING_KVP_CMD       0x1B    // 传入的KVP类型消息
#define AF_INCOMING_GRP_KVP_CMD   0x1C    // 传入组KVP类型消息

//＃define KEY_CHANGE 0xC0 //按键事件

#define ZDO_NEW_DSTADDR           0xD0    // ZDO收到了此应用的新DstAddr
#define ZDO_STATE_CHANGE          0xD1    // ZDO更改了设备的网络状态
#define ZDO_MATCH_DESC_RSP_SENT   0xD2    // 已发送ZDO匹配描述符响应
#define ZDO_CB_MSG                0xD3    // ZDO传入消息回调
#define ZDO_NETWORK_REPORT        0xD4    // ZDO收到网络报告消息
#define ZDO_NETWORK_UPDATE        0xD5    // ZDO收到网络更新消息
#define ZDO_ADDR_CHANGE_IND       0xD6    // ZDO被告知设备地址更改

#define NM_CHANNEL_INTERFERE      0x31    // NwkMgr收到频道干扰消息
#define NM_ED_SCAN_CONFIRM        0x32    // NwkMgr收到ED扫描确认消息
#define SAPS_CHANNEL_CHANGE       0x33    // 存根APS更改了设备的通道
#define ZCL_INCOMING_MSG          0x34    // 传入的ZCL基础消息
#define ZCL_KEY_ESTABLISH_IND     0x35    // ZCL密钥建立完成指示
#define ZCL_OTA_CALLBACK_IND      0x36    // ZCL OTA完成指示

//为应用程序（用户应用程序）保留的OSAL系统消息ID /事件
// 0xE0 ?0xFC
```

 在了解了case都可以用什么表示之后，我们再来看看对应情况的应对函数。在示例中总共涉及了三个函数

```c
      case ZDO_CB_MSG:
        Smart_home_ProcessZDOMsgs( (zdoIncomingMsg_t *)MSGpkt );
        break;
          
      case KEY_CHANGE:
        Smart_home_HandleKeys( ((keyChange_t *)MSGpkt)->state, ((keyChange_t *)MSGpkt)->keys );
        break;

      case AF_INCOMING_MSG_CMD:
        Smart_home_ProcessMSGCmd( MSGpkt );
        break;
```

这几个函数都是在`Smart_home.c`中定义的，都是对应情况下的希望对应解决方法，其中`Smart_home_HandleKeys`是需要用户自行实现填充的按键函数。

那么现在我们已经通过代码得知，我们所有的任务只要通过switch语句来处理网络包传输中表示事件的各种event，就可以成功的利用自写函数来控制设备进行反应。也就是说，我们已经掌握了数据接收处理的方法，那么就产生了一个更有趣的话题：数据的发送究竟是怎么完成的呢？又是通过什么样的代码来实现的呢？

## 3.数据的发送

可以肯定的是，数据的发送也是ZigBee提供好的，只需我们直接调用就好。那这个函数在哪里呢？

经过查找，我在AF.c中找到了这个函数，下面就来看看这个名为`afStatus_t AF_DataRequest`的函数

这个函数比较长，所以从函数形参开始一点一点看，我会直接将函数说明标识在形参中

```c
uint8 AF_DataRequestDiscoverRoute = TRUE;
/* 函数功能，信息发送和多路传输*/
afStatus_t AF_DataRequest( afAddrType_t *dstAddr,   //完整的ZB目标地址：Nwk地址+端点
                           endPointDesc_t *srcEP,   //起点描述（响应或确认）
                           uint16 cID,              //由配置文件指定的有效集群ID。
                           uint16 len,              //数据字节数
                           uint8 *buf,              //指向要发送的数据字节的指针。
                           uint8 *transID,          /*指向可以修改的字节的指针，
                                                    该字节将用作msg的事务序列号。*/
                           uint8 options,           //Tx选项的有效位掩码
                           uint8 radius )           //通常设置为AF_DEFAULT_RADIUS。
/*结果及返回值
*如果成功transID 增加1
*返回 afStatus_t
  #define afStatus_SUCCESS            ZSuccess           // 0x00 
  #define afStatus_FAILED             ZFailure           // 0x01 
  #define afStatus_INVALID_PARAMETER  ZInvalidParameter  // 0x02 
  #define afStatus_MEM_FAIL           ZMemError          // 0x10 
  #define afStatus_NO_ROUTE           ZNwkNoRoute        // 0xCD 
*/
```

这个函数看起来很像计算机网络中的网络层发送函数，在这里我们比较关心的是两个数据类型：`afAddrType_t`以及`endPointDesc_t`。这两个代表的是目的地址与源地址，下面就来看看这两个地址都是什么样的数据结构。先来看`afAddrType_t`

```c
/*afAddrType_t *dstAddr,  目的地址*/
typedef struct
{
  union
  {
    uint16      shortAddr;
    ZLongAddr_t extAddr;
  } addr;
  afAddrMode_t addrMode;                 //单播组播和广播选择
  uint8 endPoint;                        //目的端点
  uint16 panId;                          // used for the INTER_PAN feature
} afAddrType_t;
```

 在这里，`addrMode`是模式选择选项，是一个枚举，定义如下

```c
typedef enum
{  
  afAddrNotPresent = AddrNotPresent,      // 按照绑定表进行绑定传输

  afAddr16Bit = Addr16Bit,                // 指定目标网络地址进行单薄传输 16位

  afAddrGroup = AddrGroup,                // 组播传输

  afAddrBroadcast = AddrBroadcast         // 广播传输

} afAddrMode_t;


enum
{
  AddrNotPresent = 0,
  AddrGroup = 1,
  Addr16Bit = 2,
  Addr64Bit = 3,                          // 指定IEEE地址进行单播传输 64位

  AddrBroadcast = 15
};
```

在`afAddrMode_t`中要特殊注意一下Addr16Bit这个变量，16Bit指的是16位的网络地址，ZigBee还有一个64位的MAC地址，由IEEE进行分配和维护。

16位的网络地址是设备加入网络后由协调器或者路由器分配的，可以在Tool目录下的`f8wConfig.cfg`中更改

然后再来看`endPointDesc_t`

```c
/*endPointDesc_t *srcEP,  源地址方*/
typedef struct
{
  uint8 endPoint;                          // 端点号
  uint8 *task_id;                          // 指向应用程序任务ID的位置的指针。
  SimpleDescriptionFormat_t *simpleDesc;   // 设备的简单描述
  afNetworkLatencyReq_t latencyReq;        // 枚举结构 必须用 noLatencyReqs 填充
} endPointDesc_t;
```

其中的`SimpleDescriptionFormat_t`、`afNetworkLatencyReq_t`定义如下

```c
typedef struct
{
  byte EndPoint;                          // EP ID (EP=End Point)
  uint16 AppProfId;                       // profile ID（剖面ID）
  uint16 AppDeviceId;                     // Device ID
  byte AppDevVer:4;                       // Device Version 0x00 为 Version 1.0
  byte Reserved:4;                        // AF_V1_SUPPORT uses for AppFlags:4.
  byte AppNumInClusters;                  // 终端支持的输入簇的个数
  cId_t *pAppInClusterList;               // 指向输入Cluster ID列表的指针
  byte AppNumOutClusters;                 // 输出簇的个数
  cId_t *pAppOutClusterList;              // 指向输出Cluseter ID列表的指针
} SimpleDescriptionFormat_t;

typedef enum
{
  noLatencyReqs,
  fastBeacons,
  slowBeacons
} afNetworkLatencyReq_t;
```

从这里可以看出来，只要设置好这几个参数之后就可以直接进行数据发送了。，具体的程序可以参见

` GenericApp_SendTheMessage`，`GenericApp_Init`

也就是说，通过简单的设置，我们就明白了数据收发的操作。但数据发送时又是如何绑定的呢？这个绑定又是如何实现的呢？请听下回分解

