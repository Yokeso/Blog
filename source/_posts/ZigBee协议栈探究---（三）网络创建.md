---
title: ZigBee协议栈探究---（三）网络创建
date: 2021-03-15 11:29:49
tags: [ZigBee,CC2530,协议栈]
---

# ZigBee协议栈探究---（三）网络创建

关于设备的链接，还需要从头说起。

在（一）中我提到过一个函数`osalInitTasks`，这个函数位于`OSAL_Smart_home.c`中。我们再把这个函数拿出来看一下：

<!-- more-->

```c
/*********************************************************************
 * @fn      osalInitTasks
 *
 * @brief   This function invokes the initialization function for each task.
 *
 * @param   void
 *
 * @return  none
 */
void osalInitTasks( void )
{
  uint8 taskID = 0;

  tasksEvents = (uint16 *)osal_mem_alloc( sizeof( uint16 ) * tasksCnt); //分配内存返回缓冲区指针
  osal_memset( tasksEvents, 0, (sizeof( uint16 ) * tasksCnt)); //设置所分配内存空间单元值0

  macTaskInit( taskID++ ); //macTaskInit(0) ，用户不需考虑
  nwk_init( taskID++ ); //nwk_init(1)，用户不需考虑
  Hal_Init( taskID++ ); //Hal_Init(2) ，用户需考虑
#if defined( MT_TASK )
  MT_TaskInit( taskID++ );
#endif
  APS_Init( taskID++ );  //APS_Init(3) ，用户不需考虑
#if defined ( ZIGBEE_FRAGMENTATION )
  APSF_Init( taskID++ );
#endif
  ZDApp_Init( taskID++ ); //ZDApp_Init(4) ，用户需考虑
#if defined ( ZIGBEE_FREQ_AGILITY ) || defined ( ZIGBEE_PANID_CONFLICT )
  ZDNwkMgr_Init( taskID++ );
#endif
  Smart_home_Init( taskID ); //用户任务初始化用
}

```

在最开始的流程分析中，我们已经分析了`Smart_home_Init( taskID );`的作用：初始化用户进程，为用户创建Task。在分析的时候也提过ZDO层，也就是`ZigBee Device Object`，这个层掌管着ZigBee设备的终端节点具体提供以下功能

+ 初始化应用支持子层，网络层。
+ 发现节点和节点功能。在无信标的网络中，加入的节点只对其父节点可见。而其他节点可以通过ZDO的功能来确定网络的整体拓扑结构已经节点所能提供的功能。
+ 安全加密管理：主要包括安全key的建立和发送，已经安全授权。
+ 网络的维护功能。
+ 绑定管理：绑定的功能由应用支持子层提供，但是绑定功能的管理却是由ZDO提供，它确定了绑定表的大小，绑定的发起和绑定的解除等功能。
+ 节点管理：对于网络协调器和路由器，ZDO提供网络监测、获取路由和绑定信息、发起脱离网络过程等一系列节点管理功能。

也就是说，我们想要探究的设备发现与绑定，ZigBee组网的过程都在ZDO层进行实现。那就进入`ZDApp_Init()`这个函数来看看吧。

```c
/*********************************************************************
 * @fn      ZDApp_Init
 *
 * @brief   ZDApp Initialization function.
 *
 * @param   task_id - ZDApp Task ID
 *
 * @return  None
 */
void ZDApp_Init( uint8 task_id )
{
  // 存储task ID
  ZDAppTaskID = task_id;

  // 初始化并存储ZDO设备短地址
  ZDAppNwkAddr.addrMode = Addr16Bit;
  ZDAppNwkAddr.addr.shortAddr = INVALID_NODE_ADDR;
  (void)NLME_GetExtAddr();  // 加载saveExtAddr指针

  // 检查手动“保持自动启动”
  ZDAppCheckForHoldKey();

  // 初始化ZDO项目并设置设备-要创建的设备类型。
  ZDO_Init();

  // 向AF注册端点描述
  // 此任务没有简单描述，但我们仍然需要
  // 以注册端点。
  afRegister( (endPointDesc_t *)&ZDApp_epDesc );

#if defined( ZDO_USERDESC_RESPONSE )
  ZDApp_InitUserDesc();
#endif // ZDO_USERDESC_RESPONSE

  // 是否启动设备
  if ( devState != DEV_HOLD )
  {
    ZDOInitDevice( 0 );
  }
  else
  {
    ZDOInitDevice( ZDO_INIT_HOLD_NWK_START );
    // LED闪烁来标识设备启动
    HalLedBlink ( HAL_LED_4, 0, 50, 500 );
  }

  // 初始化ZDO回调函数指针zdoCBFunc []
  ZDApp_InitZdoCBFunc();

  ZDApp_RegisterCBs();
} /* ZDApp_Init() */
```

在这个函数中可以看到，最重要的一步就是`ZDOInitDevice()`设备在这里开始启动，并让LED闪烁。那么我们从这里来看看设备是如何启动的：

```c
/*********************************************************************
 * @fn      ZDOInitDevice
 *
 * @brief   启动网络中的设备。该功能将读取ZCD_NV_STARTUP_OPTION（NV项目）以确定是否要
 *          恢复设备的网络状态。
 *
 * @param   startDelay - 启动设备的时间延迟（以毫秒为单位）。
 *                       此延迟增加了抖动：((NWK_START_DELAY + startDelay)+
 *                       (osal_rand()&EXTENDED_JOINING_RANDOM_MASK))
 *                       当startDelay设置为ZDO_INIT_HOLD_NWK_START时
 *                       此功能将保持网络初始化。应用可以启动设备。
 *
 * NOTE:    如果应用程序想要强制“新”加入，则应用程序在调用这个函数之前应设置在
 *          ZCD_NV_STARTUP_OPTION NV中的ZCD_STARTOPT_DEFAULT_NETWORK_STATE位。“新”加  
 *          入意味着不恢复网络设备状态。使用zgWriteStartupOptions（）设置这些选项。        
 *
 * @return
 *    ZDO_INITDEV_RESTORED_NETWORK_STATE  - 设备的网络状态为已存储
 *    ZDO_INITDEV_NEW_NETWORK_STATE - 网络状态复位
 *          这可能表示ZCD_NV_STARTUP_OPTION 无法还原, 或者没有网络状态需要还原
 *    ZDO_INITDEV_LEAVE_NOT_STARTED - 重置之前，network leave将重加入选项设为真，因此，
 *          该设备不是在网络中启动（仅一次）。下次这个函数调用它将启动。
 */
uint8 ZDOInitDevice( uint16 startDelay )
{
  uint8 networkStateNV = ZDO_INITDEV_NEW_NETWORK_STATE;
  uint16 extendedDelay = 0;

  if ( devState == DEV_HOLD )
  {
    // 如果已更新NV项目，则初始化RAM项目表
    zgInitItems( FALSE );
  }

  ZDConfig_InitDescriptors();
  //devtag.071807.todo - fix this temporary solution
  _NIB.CapabilityFlags = ZDO_Config_Node_Descriptor.CapabilityFlags;

#if defined ( NV_RESTORE )
  // 直接获取键盘以查看是否需要重置nv。
  // 按住SW_BYPASS_NV键（在OnBoard.h中定义）
  // 在引导时跳过过去的NV 复位。
  if ( HalKeyRead() == SW_BYPASS_NV )
    networkStateNV = ZDO_INITDEV_NEW_NETWORK_STATE;
  else
  {
    // 确定是否应恢复NV
    networkStateNV = ZDApp_ReadNetworkRestoreState();
  }

  if ( networkStateNV == ZDO_INITDEV_RESTORED_NETWORK_STATE )
  {
    networkStateNV = ZDApp_RestoreNetworkState();
  }
  else
  {
    // 清除NV中的网络状态
    NLME_InitNV();
    NLME_SetDefaultNV();
    // 清除NWK键值
    ZDSecMgrClearNVKeyValues();
  }
#endif

  if ( networkStateNV == ZDO_INITDEV_NEW_NETWORK_STATE )
  {
    ZDAppDetermineDeviceType();

    // 仅在加入网络时延迟-无法恢复网络状态
    extendedDelay = (uint16)((NWK_START_DELAY + startDelay)
              + (osal_rand() & EXTENDED_JOINING_RANDOM_MASK));
  }

  // 初始化设备类型的安全性e
  ZDApp_SecInit( networkStateNV );

  if( ZDO_INIT_HOLD_NWK_START != startDelay )
  {
    devState = DEV_INIT;    // 删除保持状态

    // 初始化请假控制逻辑
    ZDApp_LeaveCtrlInit();

    // 检查请假控制重置设置
    ZDApp_LeaveCtrlStartup( &devState, &startDelay );

    // 离开可能为持有态
    if ( devState == DEV_HOLD )
    {
      // 设置NV启动选项以强制执行“新”连接。
      zgWriteStartupOptions( ZG_STARTUP_SET, ZCD_STARTOPT_DEFAULT_NETWORK_STATE );

      // 通知申请
      osal_set_event( ZDAppTaskID, ZDO_STATE_CHANGE_EVT );

      return ( ZDO_INITDEV_LEAVE_NOT_STARTED );   // Don't join - (one time).
    }

    // 触发网络启动
    ZDApp_NetworkInit( extendedDelay );
  }

  // 设置广播地址掩码以支持广播过滤
  NLME_SetBroadcastFilter( ZDO_Config_Node_Descriptor.CapabilityFlags );

  return ( networkStateNV );
}
```

通过阅读可以看到，这个函数的主要目的就是计算启动延迟，并启动网络设备，主要启动设备的语句为`ZDApp_NetworkInit( extendedDelay );`那我们就来看看这个函数又做了什么

```c
/*********************************************************************
 * @fn      ZDApp_NetworkInit()
 *
 * @brief   用来启动网络加入程序
 *
 * @param   delay - 启动前需等待的毫秒数
 *
 * @return  none
 */
void ZDApp_NetworkInit( uint16 delay )
{
  if ( delay )
  {
    // 启动设备前稍等片刻
    osal_start_timerEx( ZDAppTaskID, ZDO_NETWORK_INIT, delay );
  }
  else
  {
    osal_set_event( ZDAppTaskID, ZDO_NETWORK_INIT );
  }
}
```

从这两个函数名称和形参来看，都是要将`ZDAppTaskID`的标识设置为`ZDO_NETWORK_INIT`。而被设置了flag的任务在这个时候就会被处理。接收这个处理的函数就是`ZDApp_event_loop()`，由于这个函数过长，所以只看与`ZDO_NETWORK_INIT`处理相关的部分。

```c
  if ( events & ZDO_NETWORK_INIT )
  {
    // 初始化APP并启动网络
    devState = DEV_INIT;
    osal_set_event( ZDAppTaskID, ZDO_STATE_CHANGE_EVT );

    ZDO_StartDevice( (uint8)ZDO_Config_Node_Descriptor.LogicalType, devStartMode,
                     DEFAULT_BEACON_ORDER, DEFAULT_SUPERFRAME_ORDER );

    // 返回未启动事件c
    return (events ^ ZDO_NETWORK_INIT);
  }
```

在这里又调用了ZDO_StartDevice()函数，其中

ZDO_Config_Node_Descriptor.LogicalType = NODETYPE_COORDINATOR

devStartMode = MODE_HARD

且协调器编译了ZDO_COORDINATOR

也就是说，在这里就要分为协调器的开始组建网络，路由器的辅助转发网络消息，以及终端节点的加入网络进行区别，对于我们现在来说，主要的部分就来看协调器的组建网络：

我们把这一部分建网的代码拿来看一下

```c
//This function starts a device in a network.
void ZDO_StartDevice( byte logicalType, devStartModes_t startMode, byte beaconOrder, byte superframeOrder )
{
   ......
   if ( ZG_BUILD_COORDINATOR_TYPE && logicalType == NODETYPE_COORDINATOR )
   {
     if ( startMode == MODE_HARD )
     {
       devState = DEV_COORD_STARTING;
       ret = NLME_NetworkFormationRequest( zgConfigPANID, 
                                           zgApsUseExtendedPANID, 
                                           zgDefaultChannelList,
                                           zgDefaultStartingScanDuration, 
                                           beaconOrder,
                                           superframeOrder, 
                                           false );
     }
     ......
   }
```

也就是说，组建网络的重要一环原来是在`NLME_NetworkFormationRequest()`这个函数中，很遗憾，这部分函数并不开源，但从网络查询中得知，这个函数是用来判断网络状态的，主要要分为两种状态

+ nwkStatus为ZSuccess时，也就是网络创建成功了，将设备状态`decSAtatue`设置为对应的`DEV_ZB_COORD`,然后设置事件`ZDO_STATE_CHANGE_EVENT`也就是网络状态发生转变的事件，返回到`ZD_App_event_loop()`中继续执行
+ nwkStatus为非ZSuccess时，也就是网络创建不成功，那么就将增大功率返回到`ZD_App_event_loop()`中加大功率，如果功率已经是最大了，那么就将`devState`设置为`DEV_INIT`（创建网络失败）再设置事件`ZDO_STATE_CHANGE_EVENT`，返回到`ZD_App_event_loop()`

所以，无论如何，返回值都是在上面提到的语句中执行，也就是这一条

```c
osal_set_event( ZDAppTaskID, ZDO_STATE_CHANGE_EVT );
```

那再回过来看`ZD_App_event_loop()`是怎么处理这个状态的呢？

```c
  if ( events & ZDO_STATE_CHANGE_EVT )
  {
    ZDO_UpdateNwkStatus( devState );

    // 启动时，如果设备是集中器，则执行一次MTO路由发现
    if ( zgConcentratorEnable == TRUE )
    {
      // Start next event
      osal_start_timerEx( NWK_TaskID, NWK_MTO_RTG_REQ_EVT, 100 );
    }
```

调用了`ZDO_UpdateNwkStatus( devState )`,网络状态改变,这个函数会更新和发送信息通知每个注册登记过的应用终端.我们再接着来看这个函数

```c
void ZDO_UpdateNwkStatus( devStates_t state )
{
  // Endpoint/Interface descriptor list.
  epList_t *epDesc = epList;
  byte bufLen = sizeof(osal_event_hdr_t);
  osal_event_hdr_t *msgPtr;

 ZDAppNwkAddr.addr.shortAddr = NLME_GetShortAddr();
 (void)NLME_GetExtAddr();  // Load the saveExtAddr pointer.

 while ( epDesc )
  {
    if ( epDesc->epDesc->endPoint != ZDO_EP )
    {
     msgPtr = (osal_event_hdr_t *)osal_msg_allocate( bufLen );
     if ( msgPtr )
     {
       msgPtr->event = ZDO_STATE_CHANGE; // Command ID
       msgPtr->status = (byte)state;

       osal_msg_send( *(epDesc->epDesc->task_id), (byte *)msgPtr ); //发往应用任务
     }
    }
    epDesc = epDesc->nextDesc;
  }
}
```

对SampleApp的协调器来说，这里触发应用任务SampleApp_TaskID的ZDO_STATE_CHANGE事件,看下对ZDO_STATE_CHANGE的处理:

```c
         Ccase ZDO_STATE_CHANGE:   
         SampleApp_NwkState = (devStates_t)(MSGpkt->hdr.status); //获取设备当前状态
         if ( (SampleApp_NwkState == DEV_ZB_COORD)
             || (SampleApp_NwkState == DEV_ROUTER)
             || (SampleApp_NwkState == DEV_END_DEVICE) )
         {
           // Start sending the periodic message in a regular interval.
           osal_start_timerEx( SampleApp_TaskID,
                             SAMPLEAPP_SEND_PERIODIC_MSG_EVT,
                             SAMPLEAPP_SEND_PERIODIC_MSG_TIMEOUT );
         }
         else
         {
           // Device is no longer in the network
         }
         break;
```

可以看到当协调器建立网络成功,通过回调函数触发应用任务的ZDO_STATE_CHANGE事件,最终开启定时器发送周期信息.

路由器也可以按这个思路进行分析。

最后来给一下大致的流程

main()  ->  osal_init_system()  ->  osalInitTasks()  ->  ZDApp_Init()  ->  ZDOInitDevice()  ->  ZDApp_NetworkInit  ->  触发ZDAppTaskID的ZDO_NETWORK_INIT       ->  ZDO_StartDevice()-> NLME_NetworkFormationRequest()  ->  网络建立成功ZDO_NetworkFormationConfirmCB    ->                           触发ZDAppTaskID的ZDO_NETWORK_START  ->  ZDApp_NetworkStartEvt()->触发ZDAppTaskID的ZDO_STATE_CHANGE_EVT->ZDO_UpdateNwkStatus()  ->  触发SampleApp_TaskID的ZDO_STATE_CHANGE   ->开户周期信息发送的定时器.