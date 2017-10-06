---
date: 2016-10-05 21:56
title: '[FreeRTOS学习]任务通知'
categories: "FreeRTOS"
tags: [嵌入式, FreeRTOS, c]
keyworlds: 嵌入式, FreeRTOS, c
description:
---

通知（Notify）
信号（semaphore）
*每个RTOS任务都有一个32位的通知值，任务创建时，这个值被初始化为0。RTOS任务通知相当于直接向任务发送一个事件，接收到通知的任务可以解除阻塞状态，前提是这个阻塞事件是因等待通知而引起的。发送通知的同时，也可以可选的改变接收任务的通知值*
接收任务更新通知的方法
* 不覆盖接收任务的通知值
* 覆盖接收任务的通知值
* 设置接收任务通知值的某些位
* 增加接收任务的通知值

虽然RTOS任务通知速度更快并且占用内存更少，但它也有一些限制：
* 只能有一个任务接收通知事件。
* 接收通知的任务可以因为等待通知而进入阻塞状态，但是发送通知的任务即便不能立即完成通知发送也不能进入阻塞状态

> 发送通知

1. Way 1
```
BaseType_t xTaskNotify( TaskHandle_txTaskToNotify,  
                           uint32_t ulValue,  
                           eNotifyAction eAction);  
//或
xTaskNotifyGive()
```
 eActioon 枚举变量成员以及作用如下表所示。
![](http://upload-images.jianshu.io/upload_images/3168440-b6c999084f68d80f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
参数eAction为eSetValueWithoutOverwrite时，如果被通知任务还没取走上一个通知，又接收到了一个通知，则这次通知值未能更新并返回pdFALSE，否则返回pdPASS。

2. Way2
```
BaseType_t xTaskNotifyGive(TaskHandle_t xTaskToNotify );
```
其实这是一个宏，本质上相当于xTaskNotify( ( xTaskToNotify ), ( 0 ), eIncrement )。可以使用该API函数代替二进制或计数信号量，但速度更快。在这种情况下，应该使用API函数ulTaskNotifyTake()来等待通知，而不应该使用API函数xTaskNotifyWait()。
      此函数不可以在中断服务例程中调用，中断保护等价函数为vTaskNotifyGiveFromISR()。
```
static void prvTask1( voidvoid *pvParameters );  
static void prvTask2( voidvoid *pvParameters );  
/* 保存任务句柄 */  
staticTaskHandle_t xTask1 = NULL, xTask2 = NULL;  
/* 创建两个任务，它们之间反复发送通知，启动RTOS调度器*/  
void main( void )  
{  
    xTaskCreate( prvTask1, "Task1",200, NULL, tskIDLE_PRIORITY, &xTask1 );  
    xTaskCreate( prvTask2, "Task2",200, NULL, tskIDLE_PRIORITY, &xTask2 );  
    vTaskStartScheduler();  
}  
/*-----------------------------------------------------------*/  
static void prvTask1( voidvoid *pvParameters )  
{  
    for( ;; )  
    {  
        /*向prvTask2(),发送通知，使其解除阻塞状态 */  
        xTaskNotifyGive( xTask2 );  
        /* 等待prvTask2()的通知，进入阻塞 */  
        ulTaskNotifyTake( pdTRUE, portMAX_DELAY);  
    }  
}  
/*-----------------------------------------------------------*/  
static void prvTask2( voidvoid *pvParameters )  
{  
    for( ;; )  
    {  
        /* 等待prvTask1()的通知，进入阻塞 */  
        ulTaskNotifyTake( pdTRUE, portMAX_DELAY);  
        /*向prvTask1(),发送通知，使其解除阻塞状态 */  
        xTaskNotifyGive( xTask1 );  
    }  
}  
```

> 获取通知

```
uint32_t ulTaskNotifyTake( BaseType_txClearCountOnExit,  
                           TickType_txTicksToWait );  
```
参数：
* xClearCountOnExit：如果该参数设置为pdFALSE，则API函数xTaskNotifyTake()退出前，将任务的通知值减1；如果该参数设置为pdTRUE，则API函数xTaskNotifyTake()退出前，将任务通知值清零。
* xTicksToWait：因等待通知而进入阻塞状态的最大时间。时间单位为系统节拍周期。宏pdMS_TO_TICKS用于将指定的毫秒时间转化为相应的系统节拍数。
* 返回当前通知值，为0或者为调用API函数xTaskNotifyTake()之前的通知值减1。

ulTaskNotifyTake()是专门为使用更轻量级更快的方法来代替二进制或计数信号量而量身打造的。FreeRTOS获取信号量的API函数为xSemaphoreTake()，可以使用ulTaskNotifyTake()函数等价代替。

它们的中断保护等价函数：vTaskNotifyGiveFromISR()和xTaskNotifyFromISR()。

 API函数xTaskNotifyTake()有两种方法处理任务的通知值，一种方法是在函数退出时将通知值清零，这种方法适用于实现二进制信号量；另外一种方法是在函数退出时将通知值减1，这种方法适用于实现计数信号量。

如果RTOS任务的通知值为0，使用xTaskNotifyTake()可以可选的使任务进入阻塞状态，直到该任务的通知值不为0。进入阻塞的任务不消耗CPU时间。
```
/* 中断处理程序。*/  
voidvANInterruptHandler( void )  
{  
BaseType_txHigherPriorityTaskWoken;  
   
    prvClearInterruptSource();  
   
    /* xHigherPriorityTaskWoken必须被初始化为pdFALSE。如果调用vTaskNotifyGiveFromISR()会解除vHandlingTask任务的阻塞状态，并且vHandlingTask任务的优先级高于当前处于运行状态的任务，则xHigherPriorityTaskWoken将会自动被设置为pdTRUE。*/  
    xHigherPriorityTaskWoken = pdFALSE;  
   
    /*向一个任务发送通知，xHandlingTask是该任务的句柄。*/  
    vTaskNotifyGiveFromISR( xHandlingTask,&xHigherPriorityTaskWoken );  
   
    /* 如果xHigherPriorityTaskWoken为pdTRUE，则强制上下文切换。这个宏的实现取决于移植层，可能会调用portEND_SWITCHING_ISR */  
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken );  
}  
/*---------------------------------------------------------------------------------------------------*/  
   
/* 一个因为等待通知而阻塞的任务。*/  
voidvHandlingTask( voidvoid *pvParameters )  
{  
BaseType_txEvent;  
   
    for( ;; )  
    {  
        /*等待通知，无限期阻塞。参数pdTRUE表示函数退出前会清零通知值。这显然是用于替代二进制信号量的用法。需要注意的是，真实的程序一般不会无限期阻塞。*/  
        ulTaskNotifyTake( pdTRUE, portMAX_DELAY);  
   
        /* 当处理完所有事件后，仍然等待下一个通知*/  
        do  
        {  
            xEvent = xQueryPeripheral();  
   
            if( xEvent != NO_MORE_EVENTS )  
            {  
                vProcessPeripheralEvent( xEvent);  
            }  
   
        } while( xEvent != NO_MORE_EVENTS );  
    }  
}  
```
> 等待通知

```
BaseType_t xTaskNotifyWait( uint32_tulBitsToClearOnEntry,  
                            uint32_tulBitsToClearOnExit,  
                            uint32_t*pulNotificationValue,  
                            TickType_txTicksToWait );  
```
参数
* ulBitsToClearOnEntry：在使用通知之前，先将任务的通知值与参数ulBitsToClearOnEntry的按位取反值按位与操作。设置参数ulBitsToClearOnEntry为0xFFFFFFFF(ULONG_MAX)，表示清零任务通知值。

* ulBitsToClearOnExit：在函数xTaskNotifyWait()退出前，将任务的通知值与参数ulBitsToClearOnExit的按位取反值按位与操作。设置参数ulBitsToClearOnExit为0xFFFFFFFF(ULONG_MAX)，表示清零任务通知值。

* pulNotificationValue：用于向外回传任务的通知值。这个通知值在参数
ulBitsToClearOnExit起作用前将通知值拷贝到*pulNotificationValue中。如果不需要返回任务的通知值，这里设置成NULL。

* xTicksToWait：因等待通知而进入阻塞状态的最大时间。时间单位为系统节拍周期。宏pdMS_TO_TICKS用于将指定的毫秒时间转化为相应的系统节拍数。

* 如果接收到通知，返回pdTRUE，如果API函数xTaskNotifyWait()等待超时，返回pdFALSE。
```
/*这个任务使用任务通知值的位来传递不同的事件，这在某些情况下可以代替事件组。*/  
voidvAnEventProcessingTask( voidvoid *pvParameters )  
{  
uint32_tulNotifiedValue;  
   
    for( ;; )  
    {  
        /*等待通知，无限期阻塞（没有超时，所以不用检查函数返回值）。
          其它任务或者中断设置的通知值中的不同位表示不同的事件。参数0x00表
          示使用通知前不清除任务的通知值位，参数ULONG_MAX 表示函数
          xTaskNotifyWait()退出前将任务通知值设置为0*/  
          xTaskNotifyWait( 0x00, ULONG_MAX,&ulNotifiedValue, portMAX_DELAY );  
   
        /*根据通知值处理事件*/  
        if( ( ulNotifiedValue & 0x01 ) != 0)  
        {  
            prvProcessBit0Event();  
        }  
   
        if( ( ulNotifiedValue & 0x02 ) != 0)  
        {  
            prvProcessBit1Event();  
        }  
   
        if( ( ulNotifiedValue & 0x04 ) != 0)  
        {  
            prvProcessBit2Event();  
        }  
   
        /* ……*/  
    }  
}  
```

> 任务通知并查询

```
BaseType_t xTaskNotifyAndQuery(TaskHandle_t xTaskToNotify,  
                        uint32_tulValue,  
                        eNotifyActioneAction,  
                        uint32_t*pulPreviousNotifyValue );  
```
此函数与任务通知API函数xTaskNotify()非常像，只不过此函数具有一个附加参数，用来回传任务当前的通知值，然后根据参数ulValue和eAction更新任务的通知值。
  此函数不能在中断服务例程中使用，在中断服务例程中使用xTaskNotifyAndQueryFromISR()函数。
  * xTaskToNotify：被通知的任务句柄。
  * ulValue：通知更新值
  * eAction：枚举类型，指明更新通知值的方法，枚举变量成员以及作用见xTaskNotify()一节。
  * pulPreviousNotifyValue：回传未被更新的任务通知值。如果不需要回传未被更新的任务通知值，这里设置为NULL，这样就等价于调用xTaskNotify()函数。