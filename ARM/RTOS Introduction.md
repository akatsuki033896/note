可以使用的OS通常被称作嵌入式OS(Embedded OS)或RTOS(RealTime Operating System)，在OS环境运行的软件绝大多数都采用事件驱动的 (Event Driven)风格，但RTOS种类繁多且无统一的、标准的API，也没有统一的开发环境，很多RTOS仅支持几种MCU，目前没有完全通用的RTOS

RTOS中某个事件发生所触发的任务必须在预定时间内完成（不是最快完成）

## RTOS的任务调度

借助于OS的任务调度器可以并行执行多个任务

系统启动后执行必要的初始化操作(如Bootloader)、RTOS初始化，然后创建多个任务并启动调度器，这些操作都是一次性执行的代码，RTOS调度器本身是一个无穷循环， 他在定时器中断的辅助下控制多个任务并行执行。

![多任务RTOS模型](https://theembeddedsystem.readthedocs.io/en/latest/_images/multi_task_rtos_model.jpg)

现代MCU大多数都支持RTOS，如支持特权的和非特权的工作模式、异常模式等。一般RTOS内核工作在特权模式，可以访问和管理系统内全部软硬件资源， 用户任务程序工作在非特权模式只能访问属于本任务的资源。异常模式是专门处理软硬件异常和中断服务。

使用 RTOS 编写软件的伪代码风格：

```c
 Task1 () {
    // some of the initialization code for Task1
    while(1) {
      // functional code for Task1
    }
  }
  Task2 () {
    // some of the initialization code for Task2
    while(1) {
      // functional code for Task2
    }
  }
  Task3 () {
    // some of the initialization code for Task3
    while(1) {
      // functional code for Task3
    }
  }

  main () {
    // Initialize the system hardware
    // initialize RTOS
    xCreakTask(Task1); // creat the Task1, and apend it on the list of task
    xCreakTask(Task2); // creat the Task2, and apend it on the list of task
    xCreakTask(Task3); // creat the Task3, and apend it on the list of task
    vTaskStartScheduler(); // start the scheduler of RTOS, this is an endless loop
  }
```

### 时间片轮转调度

任务调度器并行处理多任务的机制：将CPU的时间分割为多个时间片，并分配给已经创建的多个任务，每个任务占用CPU的一个时间片，当正在执行的任务的时间片消耗完毕时，调度器实施任务切换(即将正在执行的任务挂起，继续执行下一个任务)，每一个任务按分配的时间片占用CPU一定时间后被挂起，如此无穷地循环， 让我们感觉多个任务被并行执行。

时间片的分割粒度由定时器的中断/溢出周期来决定，通过对定时器溢出周期的编程配置即可改变时间片粒度。

但需要优先处理某些任务时，抢占型RTOS就很必要。抢占型RTOS每个任务不仅有自己的时间片，还有自己的优先级，正在执行的低优先级任务的时间片可能会被高优先级的任务抢占。

>[!tldr]
>支持任务驱动和多任务的软件设计方法能够将复杂的嵌入式系统软件分割成多个易于实现的简单任务软件，不仅易维护还能确保实时性。同时，任何RTOS都需要额外占用嵌入式系统有限的ROM空间和RAM空间，任务调度器需要占用CPU的时间实现任务调度和任务切换，简单嵌入式系统没必要用RTOS

## RTOS的共享资源

多个任务需要共享嵌入式系统的内存、外设是很常见的，比如两个任务都需要向同一个UART端口写入单行型字符串信息，如果这个共享资源处理不当， 我们一定会发现一个字符串被另一个字符串分割。
### 互斥

使用共享资源的每一个任务必须对预先定义的互斥变量进行查询，如果被其他任务锁定则该任务将被挂起，锁定(锁定成功即可使用共享资源)、使用完毕立即释放等互斥访问共享资源的过程。

信标、队列和邮箱等都是RTOS常用的任务间通讯方法，但不是所有RTOS都支持这些方法。
## 使用 RTOS 的嵌入式软件架构

![](https://theembeddedsystem.readthedocs.io/en/latest/_images/rtos_based_es_software_structure.jpg)

除了RTOS内核(Kernel)之外，RTOS还有一部分组件与具体的嵌入式系统MCU的架构有关。当FreeRTOS用于ARM Cortex-M系列MCU时， 我们必须做一部分代码移植(Porting)工作。



