大部分嵌入式系统是不使用 RTOS 的，因为 RTOS 也要占用 RAM 和 ROM，任务调度也要占用 CPU 时间，嵌入式 MCU 的 CPU 时钟速度也是尽可能低的。不使用 RTOS 的时候系统初始化后在一个无穷循环里顺序调用各个功能子程序即可。

![](https://theembeddedsystem.readthedocs.io/en/latest/_images/no_os_es_software_structure.jpg)

异常处理程序(Exception Handler)的执行将会打破常态的主循环，无论是否有OS，允许被响应的中断或软硬件异常发生时，正在执行的任务或子程序将被暂停， 系统先执行 EH 然后再返回被暂停的地方(称作断点)继续执行。所有优秀的嵌入式系统系统软件的EH都保持尽可能短的执行时间，譬如几个微妙的时间，以保证正常主循环的执行周期不受EH的影响。

## 中断驱动

现代MCU支持上百个中断请求源，包括外部输入中断、内部功能单元中断等， 譬如串口(UART) 接收到字符的中断和字符发送完毕的中断。当中断请求发生时，CPU立即退出轻度睡眠状态， 执行中断服务程序，然后继续进入轻度睡眠。

![](https://theembeddedsystem.readthedocs.io/en/latest/_images/no_os_es_software_interrupt_polling.jpg)





