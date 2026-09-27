
ARM Cortex-M 中的 "M" 代表 **Microcontroller**，即**微控制器**，它专为低成本、低功耗、高度嵌入式的应用而优化设计，是 ARM 公司在嵌入式系统和物联网设备领域广泛应用的核心处理器架构。ARM Cortex-M系列微内核之间确存在极大的差异，究其原因是为了解决高速CPU、存储器、低速外设之间的矛盾问题。

![](https://theembeddedsystem.readthedocs.io/en/latest/_images/arm_cortex-m0plus_structure.jpg)
<center>ARM Cortex-M0的内部功能</center>

ARM Cortex-M0和M0+是两种最精简的32位处理器。ARM Cortex-M0与ARM Cortex-M3的CPU内核都采用3级流水线(Pipeline)，而ARM Cortex-M0+的CPU内核却采用2级流水线， 因此M0+的动态功耗明显低于M0和M3。

相较于ARM Cotex-M0/M0+体系架构不大于50MHz的CPU时钟速度，M3和M4的时钟速度可达200MHz，这就意味着适合于M3和M4微内核的存储器和I/O外设必须具备与之匹配的访问速度。

![](https://theembeddedsystem.readthedocs.io/en/latest/_images/arm_cortex-m3_m4_structure.jpg)
<center>ARM Cortex-M3/M4的内部功能</center>

M3和M4微内核拥有更丰富的软件调试接口(断点、数据和指令观察点等)， 还具备专用的硬件浮点数处理单元，以及I-Cache(取指令专用的高速缓存)和D-Cache(访问数据存储器专用的高速缓存)单元。

![](https://theembeddedsystem.readthedocs.io/en/latest/_images/arm_cortex-m7_structure.jpg)
<center>ARM Cortex-M7的内部功能</center>

ARM Cortex-M7微内核几乎已接近桌面计算机系统。CPU速度的MCU完全归功于先进的半导体工艺制程和紧耦合的存储器(TCM)技术。与ARM Cortex-M4F的单精度FPU相比，M7采用双精度的FPU以满足更高精度的计算需求。另外，M7的内部互联总线和片上存储器的位宽度都达到64位，拓宽单次访问的位宽度也是提升吞吐量的一种有效方法。[Pixhawk 4 Mini](https://docs.px4.io/v1.12/zh/flight_controller/pixhawk4_mini.html)就是用M7。

## 存储器系统

存储器系统的访问速度和访问方法严重影响MCU的整体性能。

ARM Cortex-M系列微内核的存储器采用扁平化管理，使用32位的地址总线宽度意味着整个存储空间共4G(即$2^{32}$)字节，片上或片外的全部程序存储器、 数据存储器和外设等都位于这4GB空间内。ARM公司将4GB空间简单地均分为8个区并指定每个区块的主要用途、访问属性等。

![](https://theembeddedsystem.readthedocs.io/en/latest/_images/arm_cortex_m_memories.jpg)
<center>ARM Cortex-M存储系统及其分区</center>

私有外设：微内核中的特殊外设，如SysTick、NVIC、MPU、系统配置和状态、系统异常处理、系统调试和控制等。访问和配置这些私有外设相关的地址空间可以控制ARM Cortex-M系统工作模式，从 `0xE0000000` 开始的1MB空间被固定用于这些私有外设。

XN(eXecute Never)：一个存储分区是否允许执行。外设/设备区是不允许执行的，而ROM和RAM区都是允许执行的。ROM区通常用于保存程序指令。RAM访问速度比ROM快,经常把ROM的指令取到RAM然后执行. 因此系统每次启动时将ROM中的程序指令复制到RAM区，再从RAM区开始执行程序,可以把正在反复修改的程序直接写到RAM区并从RAM区执行程序不仅速度快，而且有利于节约ROM的写寿命。

外设的读操作:使用指令将某个输入外设的对应存储器地址单元的内容加载到CPU内核的某个寄存器中

外设的写操作：将CPU内核的某个寄存器内容更新到某个输出外设的对应存储器地址单元中

### 寄存器

ARM Cortex-M系列微内核属于典型的Load-Stroe架构类型，即任何操作都必须在CPU内核的寄存器间进行，而且所有外设都没有Cache， 任何的外设操作都被统一为Load(加载)和Store(存储)操作。

![](https://theembeddedsystem.readthedocs.io/en/latest/_images/arm_cortex-m_registers.png)

ARM Cortex-M的CPU内核中有16个32bit寄存器可用，此外还有几个特殊寄存器，包括3个程序状态寄存器、3个中断/异常屏蔽寄存器和1个控制寄存器。

- R0~R7：可以被16位Thmub指令操作也可以被32位的ARM指令操作
- R8~R12：只能由ARM指令操作
- R13：堆栈指针寄存器，ARM Cortex-M在物理上有两个堆栈指针寄存器：MSP(主堆栈指针)和PSP(进程堆栈指针)，任何时候R13到底对应物理上的哪个寄存器由CONTROL(控制)寄存器所控制，两个堆栈指针的结构是为了满足运行操作系统和用户进程的需要
- R14：连接寄存器(LR)，用于存储子程序或函数的返回地址
- R15：程序计数器(PC)，读R15将得到“当前正在执行的指令存储地址+4”

读取程序状态寄存器将会得到当前指令的执行结果，如是否溢出、是否进位等，设置控制寄存器将会改变系统CPU的执行模式，如启用FPU处理浮点数等。

## 内部互联总线系统

AMBA(Advanced Microcontroller Bus Architecture)最初由ARM定义用于数字半导体产品设计，成为半导体设计领域的一类片上组件互联协议标准，与其他计算机系统总线并无本质区别

## 引脚

每一个MCU芯片都有若干个I/O引脚，极少数片上外设不需要引脚，如定时器等不需要占用芯片的外部引脚，其他外设都需要通过I/O引脚与MCU外部的功能单元或电子元件连接。

当某个嵌入式系统需要使用一种小型的LCD显示器时，我们必须占用MCU的几个I/O引脚和内部的SPI或I2C外设接口将显示器电路单元CPU连接起来，然后编程将某些寄存器中的数据写入LCD显示器的RAM中，这样的接口电路设计实际上是将LCD的RAM写端口映射 到嵌入式系统的SPI或I2C外设的某个或某些存储器空间。

