# System on Cat设计文档

计52 王纪霆 2015011251

计52 于志竟成 2015011275

计52 魏钧宇 2015011263

## 项目简介

System on Cat及其附属工程中，我们实现了：

+ 基于chisel3的五段流水线RV32IA SoC，支持中断、MMU，实现了多路组相联Cache
+ C语言实现的RISC-V版监控程序，并成功被其他两组分别移植使用
+ 将uCore+ on RV64G移植至SoC上并正确运行
+ 完成Decaf语言的交叉编译器，能够将Decaf高级语言编译成RISC-V 32汇编，并在uCore+调度下运行
+ 初步在硬件上支持双核，并将含smp的uCore+移植至SoC上，可以部分正确运行。

## 整体架构

整体硬件架构图可见[最终展示报告](http://os.cs.tsinghua.edu.cn/oscourse/csproject2018/group01?action=AttachFile&do=view&target=%E6%9C%80%E7%BB%88%E5%B1%95%E7%A4%BA%E6%8A%A5%E5%91%8A.pdf) P4, 有清晰的图示。

硬件大致由以下几部分组成：

+ 数据通路/Datapath，包含五段流水线，以及负责向MMU发出请求的部件；
+ MMU，负责虚拟地址到物理地址的转换，包含TLB, PTW, Cache等部件；
+ 总线/SysBus，负责调度两个核发出的请求，以及驱动外设；
+ 中断模块，包括CSR, Client, PLIC等部件

各模块的详细构成将在以下详细叙述。

## 模块设计-单核部分

### 流水线

我们选用了最常见的五段流水线架构。各段的功能如下：

- IF：从总线读出指令
- ID：译码，由指令获得控制信号、读出所需的寄存器值、对立即数进行符号/无符号扩展；对跳转指令进行预测
- EX：将操作数送入ALU进行运算，获得分支指令结果
- MEM：读/写总线
- WB：写回寄存器，读/写CSR寄存器（具体处理流程见“中断与异常”一节）

由于流水线功能和逻辑比较复杂，以下按照修改与演进过程叙述其处置细节。

#### 基本流水线

基本流水线即实现基本的指令与功能，但不包括中断、异常、MMU等。这一部分逻辑全部实现之后，即可完整运行关闭所有编译选项的监控程序。

##### 控制器

以下为使用的控制信号及其功能：（参考了riscv-mini和rocket-chip的设计）

| 控制信号名      | 位数   | 介绍                       |
| ---------- | ---- | ------------------------ |
| `rxs1`     | 1    | 该命令是否使用rs1               |
| `rxs2`     | 1    | 该命令是否使用rs2               |
| `A1_sel`   | 2    | ALU操作数1的来源（rs1/PC）       |
| `A2_sel`   | 2    | ALU操作数2的来源（rs2/立即数/4）    |
| `imm_sel`  | 3    | 立即数类型                    |
| `alu_op`   | 4    | ALU运算类型                  |
| `jal`      | 1    | 是否为JAL指令                 |
| `jalr`     | 1    | 是否为JALR指令                |
| `mem`      | 1    | 是否需要访存                   |
| `mem_cmd`  | 3    | 访存操作类型（读/写/LR/SC/AMO）    |
| `mem_type` | 3    | 访存字节类型（B/H/W/BU/HU/WU）   |
| `wb_en`    | 1    | 是否需要写回                   |
| `wb_sel`   | 2    | 写回数据来源                   |
| `csr_cmd`  | 3    | CSR访问类型（Write/Set/Clear） |
| `legal`    | 1    | 是否为合法指令                  |
| `fence_i`  | 1    | 是否为FENCE.I               |
| `fence`    | 1    | 是否为FENCE（未使用）            |
| `mul`      | 1    | 是否为MUL（未使用）              |
| `div`      | 1    | 是否为DIV（未使用）              |
| `branch`   | 1    | 是否为B系列跳转指令               |
| `amo_op`   | 4    | AMO原子操作类型                |

RISC-V的每一条指令，都最多涉及三个寄存器：两个源寄存器(rs1, rs2)和一个目标寄存器(rd)，而这三个寄存器编号在指令中所处的位置是固定的，因此控制信号中只包含了`rxs1`, `rxs2`, `wb_en`这样的使能信号，而无需将寄存器编号单独记录下来——需要使用时直接在原始指令里提取即可。

##### ALU

ALU本身的功能并不复杂，但参考了rocket-chip的处理：ALU同时负责两种功能，既负责完成普通的算术运算，也负责产生分支跳转条件是否满足的信号。在实现中，这两个信号是分开产生的，由于分支跳转条件判断比较简单，路径延迟也较短，该信号可以更快地产生，以便尽快完成接下来的分支逻辑。

##### 旁路

旁路逻辑较为简单，即ID段需要读出的寄存器正在被EX/MEM/WB段的指令修改，还未写回到寄存器中，此时就不读寄存器，而是直接拿之后段待写回的数据作为读出的结果。具体实现上同样参考了rocket-chip的一个优化，由于ID段功能简单，其延迟显著小于需要算术运算和判断分支的EX段，因此应当尽可能地将EX段的一部分功能搬到ID段来实现：

在ID段，我们提前判断是否会发生数据旁路（RAW冲突），如果会，那么同时判断出下一个周期EX段应当从何处取得数据，并存在段间寄存器中，供下一个周期使用，这样便减小了EX段的工作量。

##### 阻塞

当段需要等待其他段的运行结果时，即需要阻塞，保持其段间寄存器不变。具体来说，包括以下情况：

- IF段与MEM段同时访存时，只有MEM段可以得到结果，IF段应被阻塞；
- ID段需要读出的寄存器将要被处于EX段的Load指令修改，此时由于数据直到MEM段才能被真正读出，所以需要阻塞等待。

##### 跳转

我们实现了RISC-V文档中推荐的简单静态分支预测：对B系列指令，要向PC增加方向跳转时预测不跳转，向PC减小方向跳转时预测跳转。

实际上，由于B系列指令的跳转目标是PC+符号扩展立即数的形式，因此只需判断立即数最高位是0还是1，即可知晓跳转方向，这使得判断十分简单。

不过，考虑到IF段要经过MMU和Cache，可能延迟较大，我们没有在IF段进行预测，而是在ID段译码完成后，根据译码出的结果判断是否为B指令、是否应该跳转，并计算出跳转地址，同时将IF段的指令清除。这样，即使预测成功，也还是会损失一个周期的运行时间。

究竟预测是否正确，则是在EX段判断的。若之前预测错误，则清除流水线中的指令，并跳转到正确的目标地址。故若预测失败，则会损失两个周期运行时间。

##### 访存

IF段、MEM段分别与IFetch, DMem两个元件相连接，分别负责指令与数据的访存。这两个元件分别与SysBusConnector元件相连接，该元件将指令、数据访存请求进行统筹，当发生冲突时通知指令段阻塞等待。

MMU与总线的时序是基本相同的，因此同样的访存逻辑即可对MMU使用，也可直接接到总线上。访存的接口设计比较特殊：需要提前一周期将请求送入器件中。以下做详细解释：

例如，若MEM段需要写入一个数据，则访问的地址、需要写入的数据必须在EX段时就准备好，当时钟上升沿到来时，指令从EX段进入MEM段，而DMem、SysBusConnector、MMUWrapper(或者总线上的元件)三个元件也同时用内部寄存器存下这个访存请求，直到访存结束为止都不会被修改。这样，即使这个访存请求需要多周期完成，请求也可以保证不会被打断。

#### 中断处理

为了加入中断处理，流水线也需要进行相应的修改以完成流程控制。这一部分实现完成后，应可以运行带中断的监控程序。

##### CSR

CSR作为一类特殊的寄存器（具体内容及功能见“中断与异常”一节），与普通寄存器有很大不同。其值直接影响到整条流水线中所有指令的执行（如MMU需要的页表基地址、中断使能设置等都存放在CSR中），因此可能会产生WAR冲突。同时，CSR也会因流水线之外的外部信号产生变化（如时钟中断、外部中断等）。最后，CSR需要处理精确异常，当某条指令发生异常时，需要保证该指令之后的所有指令都不能被执行。

因此，为了尽量减少“CSR已被外部修改，但指令却未及时读到最新数据”的情况，以及消除WAR冲突，并且正确处理异常，CSR的读写被我们放在了WB段。并且，用以下额外措施来保证运行的正确性：

- 由于是在WB段读出数据，指令始终可以保证其读到的是最新的数据。
- 若要写入的CSR立即被读出使用，即产生RAW冲突，读CSR的该指令需被阻塞等待，直至CSR写回完成。
- 若写入的是敏感的CSR（如控制分页机制的`satp`），则需要将流水线中所有其他指令全部清除。事实上，这一机制需要该指令在MEM段时就提前进行判断，这样可以保证该指令在WB段修改CSR时，位于MEM段的指令已被清除，不进行访存。
- 若当前的指令为xret/ecall/ebreak/其他会导致异常的指令，则必然产生跳转（或是因产生异常而跳转，或是因xret而需跳回xepc处），在跳转前，也需要将流水线中的其他指令清除。

#### 完整流水线

最后，流水线在加入MMU之后还需要一次较大的改动。这一次修改结束后，就已是最终实现结果了。

##### MMU

MMU带来的直接结果是某些访存需要多个周期才能结束。在这种情况下，阻塞信号将从MMUWrapper（见“MMU模块”一节）向上传至SysBusConnector、IFetch/DMem，再传到流水线中，使得整个流水线同时阻塞住，等待访存完成。

与此同时，由于一些指令需要多个周期才能完成，中断机制也需要被修改。在多个周期的访存过程中，有可能其中某一刻传来了仅维持一周期的中断信号，这个中断信号需要被保存下来，等到当前访存结束后再做处理。

另外，与MMU配套的还有一个用于TLB flush的指令SFENCE.VMA，该指令执行时自然也需要清空流水线。

##### 原子操作

原子操作的具体实现见“原子操作”一节，此处仅说明其与流水线的交互情况。实际上，原子操作是在DMem中实现的，对上层的五段流水线而言，MEM段是因原子操作而阻塞，还是因MMU在填写页表而阻塞，抑或是因Cache在写回而阻塞，都没有区别。因此，原子操作并不需要对流水线逻辑进行修改。

### 中断和异常

中断和异常的处理是构建计算机系统的重要环节，广义上的中断包括了异常、时钟中断、软件中断、硬件中断四个类型，SystemOnCat中断处理模块基于Risc-V文档中要求的中断处理框架，下面介绍具体的实现方法

#### 处理器的特权级

根据Risc-V文档，Risc-V框架共支持四个特权级（M，S，H，U），其中M特权级为Machine特权级，是Risc-V文档唯一要求必须实现特权级，S特权级为Supervisor特权级，文档建议操作系统应当运行在当前特权级下，H特权级为HyperSupervisor特权级，适用于大型分布式系统，可支持运行多个操作系统，其中有一个超级管理员作为一系列操作系统的管理者，U为User特权级，是用户程序运行的特权级。

Risc-V文档针对不同的系统给出了不同的特权级实现建议，针对小型的嵌入式系统建议只需要实现M特权级即可，针对可运行可信操作系统的情况，建议实现M+U特权级，其中的操作系统直接运行在M特权级下，应用程序运行在U特权级下，针对运行不可信操作系统的情况，可以实现M+S+U特权级组合，不可信的操作系统执行在S特权级下，不能够通过修改M特权级的寄存器来破坏整个机器的安全性，对于更加高级的系统可以实现M+S+H+U的组合，整个框架具有很好的可扩展性，在SystemOnCat中，我们的uCore操作系统是可信操作系统，所以我们实现的是M+U组合的特权级，此外由于不支持用户层级的中断，因此没有实现U特权级下的一系列中断寄存器

Risc-V文档中没有指明当前特权级应当保存的具体位置，只在Status寄存器中有一个用来保存中断之前特权级的标记位，特权级的切换通过修改这个标记位，然后执行ERET来实现，为了方便，我们自己增加了PRV寄存器用于保存当前特权级，但该寄存器不允许外部修改

#### CSR寄存器

CSR寄存器记录处理器当前的状态，中断异常的返回地址，入口地址，中断原因，中断值等一系列信息，按照Risc-V框架的设计，每一个特权都有一系列自己的CSR寄存器用来保存该特权级下的中断信息，由于我们仅支持系统级的中断，因此只实现了M特权级对应的CSR寄存器，以下简单介绍之：

MStatus：
用于记录当前各个特权级下的全局中断是否使能（mie位），中断之前的特权级（mpp位）

SATP:
原本是一个S特权级的寄存器，但是由于我们操作系统是可信的，因此将其实现成为了一个M特权级下的寄存器，主要和虚拟存储管理相关，内部有mode位用于记录是否开启虚拟存储映射，asid用于记录当前的进程号，ppn用于记录一级页表的物理页号

MIE：
用于记录各类中断（包括时间中断，软件中断，硬件中断）是否使能

MIP：
用于记录各类中断是否在等待，即是否有中断还没有处理

MTVEC：
用于记录中断处理的入口地址

MEPC：
用于记录中断返回地址epc

MCAUSE:
用于记录中断的原因

MHARTID：
用于记录当前核的ID，主要用于实现多核

上述各类寄存器都是可读的，部分是可读可写的，CSR的读写在WB段，和流水线交互的部分有很多旁路逻辑需要处理，这部分已在流水线部分详细介绍

#### 广义中断的处理流程

正如前文所述，广义中断包括四个类型，他们的共性在于虽然触发源不同，但是处理和恢复的过程都很类似

进入中断时：

1. 流水线在MEM段结尾处就已经知道中断的发生，这时需要清空之前流水段中的指令
2. 在WB段，CSR寄存器将会记录此时的epc（对于异常来说，epc指向触发异常的指令，对于中断而言，如果触发中断时的指令是一条跳转指令，epc返回它自身，否则返回它的下一条指令），如果中断使能（status中的全局使能位有效且mie中的局部使能位有效）则进行如下步骤，否则不处理当前这个中断
3. 在mcause中记录触发中断的原因
4. 在mtval中记录中断的相关数值（比如缺页异常的地址，指令异常的错误指令）等
5. 关闭全局中断，将原有的mie保存到mpie中，设置mie为false
6. 记录中断前的特权级，将当前的特权级设置为M特权级
7. 跳转到mtvec中记录的中断处理入口地址处开始中断处理，处理过程中将根据mcause的具体中断原因来执行相应的中断处理代码

退出中断时：

1. 恢复到中断前特权级
2. 将mie恢复为mpie
3. 跳转到mepc处执行

四类广义中断的优先级：
异常 > 外部中断 > 软件中断 > 时钟中断

#### 异常种类及其处理流程

目前支持的异常主要有以下几类：

1. 非法指令
2. 指令/数据地址不对齐
3. 三类缺页异常（包括存储缺页，读取缺页，取指缺页）
4. ECall，类似于系统调用

出现异常之后，流水线将保证触发异常之前的指令都能够正确执行结束，触发异常以后（包括有异常的指令）都没有执行。接下来按照广义中断的处理流程进行处理

#### PLIC模块（硬件中断的触发流程）

在多核处理器系统中，硬件中断有多个中断源（多个外部设备），对应有多个中断目标（两个核），PLIC模块主要用于将多个中断源的中断请求整合成一个中断信号送给相应的中断源等待处理，以下是一个外部中断的处理流程：

1. 当一个外部中断到来后，它会将自己的中断闸门关闭，在处理完这个中断之前不会接受下一个中断，同时设置ip寄存器为true，以防这个中断信号出现丢失
2. 当ip寄存器为高时，PLIC模块会修改中断目标的IR寄存器，在IR寄存器中记录了中断原因，同时会将ext_irq_r设置为高，通知两个中断处理器
3. 处理器会通过访存指令读自己的IR寄存器（地址映射的规则参见总线设计文档），获知中断原因，读IR寄存器将被视为响应中断，IP寄存器将被清空
4. 处理器处理完成以后会向IR寄存器中写入中断号，写操作被视为中断处理结束，中断闸门将被打开

#### Client模块（时钟、软件中断的触发流程）

Client模块主要用于触发软件和时钟中断，它是一个总线设备，内部有一系列可用于读写寄存器，针对他们的读写都通过访存指令实现（地址映射的规则参见总线设计文档）
Client模块包括的寄存器主要以下几个：

1. msip：用于触发软件中断，软件向这个位置写1将触发中断，写0将消除中断
2. cmpl, cmph：两个寄存器用于存储触发时钟中断的比较值，两个寄存器分别记录比较值的低位和高位
3. tmel, tmeh：两个寄存器类似于计数器，用于存储时钟中断当前的数值，当tmeh和tmel拼接以后得到的数值大于等于cmph和cmpl拼接之后的数值时将触发时钟中断

Client每一个周期将给（tmeh:tmel）加1，当触发软件中断时sft_irq_r将被设置为高，当触发时钟中断时tmr_irq_r将被设置为高。



### 总线

#### 总线协议

为了使CPU和外设之间能够以一种高效、统一的接口进行数据的交换，我们参考Wishbone总线为System on Cat专门设计了一套总线协议。在设计的过程中，我们尽可能地在协议的简洁与高效之间保持最好的平衡。由于参考了Wishbone总线的设计，我们的总线协议与Wishbone之间有较多的共同之处。二者之间应当也存在不少差异，这主要是由于

1. 限于时间和精力，我们无法在课程期间完整阅读、消化和实现Wishbone总线的规范
2. Wishbone总线一些特性，例如地址标记、周期标记等，在我们的系统中并没有用武之地

我们最终得到的总线协议具有如下几个特点：

1. 允许上一次总线交互结束的同时接受下一次交互请求，中间无时间间隔
2. 所有状态的改变和数据的交换都在统一的时钟上升沿发生
3. slave处理请求所需的周期数任意，无需预先确定

为了方便后续的介绍，我们有必要在此对即将用到的一些关键概念进行说明：

- master, slave：指连接到总线上的两种不同的设备。master指能够主动发出总线请求的设备，slave则指能够被动接受总线请求的设备。在System on Cat中，master包括CPU中的每个核，而slave包括了RAM、Flash、串口、ROM、PLIC等。

下面我们将从接口信号、连接结构及交互时序三个方面对System on Cat的总线协议进行介绍。

##### 接口信号

Master/Slave信号如下表所示：

| 信号名称      | 类型         | 方向   | 说明                                    |
| --------- | ---------- | ---- | ------------------------------------- |
| `clk`     | `Bool`     |      | 总线时钟                                  |
| `cyc_o`   | `Bool`     | M>S  | 总线占用请求                                |
| `stb_o`   | `Bool`     | M>S  | 总线交互请求，为高时表示master设备请求和slave设备进行一次交互  |
| `dat_o`   | `UInt(32)` | M>S  | master设备传给slave设备的数据                  |
| `adr_o`   | `UInt(32)` | M>S  | 总线交互的地址                               |
| `sel_o`   | `UInt(4)`  | M>S  | 数据的字节掩码                               |
| `we_o`    | `Bool`     | M>S  | 为高时表示请求的交互为写类型                        |
| `stall_i` | `Bool`     | M<S  | 为高时表示slave未能接受交互请求                    |
| `ack_i`   | `Bool`     | M<S  | 为高时表示slave顺利完成了交互请求的处理                |
| `err_i`   | `Bool`     | M<S  | 为高时表示slave在试图处理交互请求的过程中遇到了错误 （实际上没用到） |
| `rty_i`   | `Bool`     | M<S  | 为高时表示slave示意master重试 （实际上没用到）         |
| `dat_i`   | `UInt(32)` | M<S  | slave设备返回的数据                          |

##### 连接结构

System on Cat的总线采用了shared bus逻辑连接结构，可以看成一条信道上接入了若干master和slave设备，在任何时刻最多支持一对master和slave之间的总线通信。我们在设计的时候还考虑过crossbar switch结构，即每对master和slave设备之间都有单独的信道，对于任意一队master和slave设备，只要它们均是空闲的，就可以进行总线通信。我们最终在shared bus和crossbar switch两种连接结构中选择前者主要是为了将复杂程度降低到可控的范围内以减小实现过程中的风险。

由于总线上允许接入的master和slave均可以是多个，但每个时刻只允许各有一个活跃的设备，我们需要在总线中加入对master设备进行仲裁和对slave设备进行选择的模块，我们将它们分别称为SysBusArbiter和SysBusTranslator。有趣的是，这两个模块也可以被分别看做master设备和slave设备：SysBusArbiter的作用相当于将若干master设备合并为一个master设备，而SysBusTranslator相当于将多个slave设备合并为一个slave设备。以这个视角看整个总线的设计，我们就可以将它解耦为三个部分：

1. SysBusArbiter：master设备的仲裁
2. SysBusTranslator：slave设备的选择
3. 单对master和slave之间的交互时序

下面我们将对以上各部分分别进行更详细的介绍。

##### 交互时序

时序方面我们参考了Wishbone的pipeline模式。在一对master和slave设备之间的总线交互总是由master发起，slave结束，所有设备均由统一的总线时钟进行同步。每个交互的过程大致可以分成两段：

1. master准备阶段：master准备好接口信号，等待slave接收进行处理。
2. slave处理阶段：slave已经接收master的请求，正在进行处理。此时master需要等待slave返回处理结果

顾名思义，pipeline模式有类似流水线的方式允许上面两个阶段同时进行。如下所示：

```
MASTER(1)    SLAVE(1)
             MASTER(2)    SLAVE(2)
                          MASTER(3)    SLAVE(3)
```

即在slave正在处理一个请求的时候，master就可以准备好下次交互的信号，等到slave处理完这次请求的同时就可以立即开始处理下一个请求。

两个阶段的开始和结束均在总线时钟的上升沿，两段的结束分别由slave的`stall_i`和`ack_i`/`rty_i`/`err_i`信号标志。master/slave设备间的一次完整交互先后经过了如下事件：

1. master将请求相关信号（包括`adr_o`,`dat_o`等）设置好（保持），将`cyc_o`和`stb_o`拉高，开始等待slave的响应。
2. slave能接受该请求时，将`stall_i`拉低并保持到下一个时钟上升沿，在该时钟上升沿后，slave开始处理请求，master的请求信号可以不再保持。若有下一个请求，master可以准备下一个请求的信号。
3. slave处理结束，将`stall_i`拉低，同时将`ack_i`（或者`err_i`、`rty_i`）置高，`dat_i`设为处理返回的数据。上列信号的状态均需要保持到下一个时钟上升沿。如果此时master还有请求（即`cyc_o`与`stb_o`为高），则下一个时钟上升沿处就可以接受该请求。

此时看上去`stall_i`在有`ack_i`、`err_i`、`rty_i`的情况下似乎是完全多余的（`stall_i`为低似乎当且仅当`ack_i`、`err_i`、`rty_i`三者之一为高），但是实际并非如此。它们有完全不同的含义：`stall_i`标志的是下一次交互在slave中处理的开始，而另外三个信号标志的是上一次交互的结束。在只有一个master和一个slave的情况下，一次交互的结束绝大多数情况下都是下一次交互的开始（除了连续一串交互中的第一个），但是在有多个master的情况下，一个master设备在收到上一次请求的反馈时并不能保证总线能处理下一次请求，因为下一次总线使用权可能被分配给了其他master。

#### SysBusArbiter

对slave来说，SysBusArbiter对多个master的请求进行仲裁，并根据仲裁结果将其汇总为一个请求，而对各个master来说，SysBusArbiter就是一个slave。

仲裁方面，SysBusArbiter采用了Round-Robin策略，根据当前总线被占用的情况（设当前占用总线的master为`current`）和各个master的总线占用请求（`cyc_o`信号）计算出下一次总线使用权应该交给哪个master（设这个master为`next`）。

在合并master上以及分派slave返回的信号上，SysBusArbiter遵循之前的讨论：`dat_o, we_o, sel_o`等是下一次交互所需的信号，因而从`next`接过来，`stall_i`标志下一次交互在slave中处理的开始，因而接到`next`上，而`ack_i`等信号标志当前交互的结束，因而接到`current`上。

当`ack_i`、`err_i`和`rty_i`中有一个为高时，SysBusArbiter在下一个时钟上升沿处将`current`置为`next`，表示总线分配给下一个master使用。

#### SysBusTranslator

SysBusTranslator和SysBusArbiter其实基本上是对称的。不同之处在于，下一次交互的slave不是像master那样通过仲裁决定，而是直接通过地址转换计算得到。

### 外设

外设（总线上的slave设备）的地址映射表如下：

| 地址                     | 说明            |
| ---------------------- | ------------- |
| `00000000`-`003fffff`  | RAM1          |
| ` 007ffff8 `           | 串口数据          |
| `007ffffc `            | 串口缓冲区信息       |
| `007fffc0 `-`007fffdf` | IRQ Client寄存器 |
| `007ffff0 `-`007ffff7` | PLIC          |
| `007fffe0 `-`007fffef` | Flash         |
| `ffffffc0`-`ffffffff`  | ROM           |
| 其他                     | RAM2          |

由于chisel不支持三值逻辑，而系统与外设进行交互却又需要`inout`类型的接口，我们不得不在系统的外围加一层用verilog实现的外设模块，然后在系统中通过这些模块提供的slave接口来实现对外设的控制。接下来我们将分别对这些外设模块进行介绍。

#### 串口

由于串口的传输速率相对CPU的执行速率来说十分缓慢，我们在串口模块处设置了各15字节的输入和输出缓冲区。缓冲区中包含的数据长度可通过从`007ffffc `读取一个字节得到。这个字节的低四位和高四位分别表示输入缓冲区的包含的字节数和输出缓冲区还能容纳的字节数。在系统试图从串口读数据时，如果输入缓冲区非空，则可以在一个总线周期内得到数据，否则串口模块会等待串口收到下一个字节时再返回`ack_i`信号。同理，在系统试图向串口写入数据时，如果输出缓冲区未满，则将该数据放入输出缓冲区尾部即可，否则串口模块需要等待输出缓冲区至少排掉一个字节。

我们这种引入输出缓冲区的设计会有一个负面的影响：串口数据的发送和CPU指令的执行成为了异步的关系，即我们执行写串口指令之后的某个无法预知的时刻写入串口的数据才会被真正发送出去。这在我们的系统中不是什么问题，但是在某些对同步性有较高要求的应用场景中，这可能就会成为一个很严肃的问题。不过这也有个以增加接口复杂度为代价来弥补问题的方法：再给串口添加一个等待输出缓冲区排空的接口，仅在需要同步的时候使用即可。这个解决方案在软件实现的缓冲区里也有应用，例如C、Java、Python等语言的标准库都提供了`flush`这个接口。

#### RAM

RAM的实现非常直接，没什么可说的，基本上就是直接套一下，把信号换个名字传过去就可以了。

#### Flash

Flash在我们的系统中算是外存，也就是说

1. 我们期望它的容量十分大，可以远大于地址空间（尽管实际上不是如此）
2. 它的访问的频繁程度远低于RAM

因此，我们采用了先写入Flash地址再进行实际访存操作的设计。这样，Flash模块以多一次总线交互为代价，将占用的物理地址数从Flash的大小降低到十几个（差不多是取了`log`）。不过这样处理实际上会带来一个潜在的问题：硬件无法保证Flash操作的原子性。换句话说，指令流可能在对Flash进行操作的中间（即写入地址和进行实际访存之间）被打断，而这样显然会导致一些我们不期望得到的后果。经过一些思考，我们认为这个问题最合理的解决办法是将加锁以保证原子性这件事留给软件来完成，硬件不做任何特殊的处理。

#### ROM（MasterCat）

ROM是一组存储只读数据的存储单元。我们在系统中加入ROM主要是为了系统引导的方便：在复位后系统需要能够自动进行一些可控的初始化操作，但用硬件实现过于复杂，用软件实现又存在应该由谁将软件加载进RAM的问题，因此我们选择使用ROM来存放执行初始化工作的代码（命名为MasterCat，这相当于介于硬件和软件之间的“固件”），然后只需要将系统的初始PC置为ROM的起始地址，就可以使系统复位后自动进行预设的初始化工作。

我们将MasterCat实现得尽可能简单，以减小ROM的容量，降低硬件的复杂度。它完成的工作可以说就是尽快甩锅：它从Flash中读取前4096个字节（包含接下来要执行的代码）加载到RAM中，并跳转到这段数据的起始地址（0）。

#### 其他

PLIC和IRQ Client是与比较特殊的总线设备，用于提供与中断和异常机制相关的配置接口。出于某种考虑，RISC-V规范并没有将这些接口设计为普通的CSR，而是采用了地址映射的机制。



### MMU

MMU（Memory Management Unit）模块能够完成虚实地址之间的转换，通过虚存管理机制，实现了不同进程之间的地址隔离

#### MMU概览及顶层模块MMUWrapper的设计

在SystemOnCat中MMU主要由TLB，PTW两个模块组成，它向上和综合流水线的两个访存段的Connector部件相连，这个Connector部件能够协调来自IF和MEM两个段的访存请求，同时还需要和另一个核的Connector交互，从而方便原子指令的实现；它向下和Cache部件相连，所有的访存通过Cache，然后Cache再和实际挂载在总线之上的Ram交互，Cache及其一致性的实现参见Cache设计文档

MMU的顶层封装模块称为MMUWrapper，这个模块将tlb和ptw实例化，内部有一个状态机，能够支持地址转换和实际访存两种状态的切换：

1. 状态1:进行地址转换，查找tlb需要一个周期，如果tlb发现自己缺失，它会去ptw中进行页表访存，此时将阻塞住上层mmu的访存过程，mmu保持在状态1，如果转换顺利完成进入状态2开始进行实际的访存，否则出现缺页异常通知给更上层的流水线
2. 状态2:一旦地址转换顺利完成，就通过拿到的地址进行内存的访问

#### TLB 的设计实现

TLB是页表的缓存，在我们的设计中，TLB内含一个小型的状态机，共包含两个状态（`ready`, `request`）：

1. `ready`态表明当前的tlb能够接受一个新的地址转换请求，在`ready`态的tlb一旦接收到一个tlb请求，就会首先去查看tlb entry，看这个请求中包含的物理地址是否已经在tlb表中，查找的过程为直接映射，物理地址的低12位作为页内偏移，中间的5位作为index，剩下的部分连同asid作为tlb表的tag，每当找到一个对应的tlb entry时，就会对比它的tag是否和当前缓存的tag一致，如果一致就说明tlb hit，否则说明tlb misstlb hit不会导致状态的切换，tlb miss将使得tlb内的状态机切换到`request`态
2. `request`状态下，tlb会向更底层的ptw发出tlb refill请求，这一请求将被ptw解析为一系列的访存操作，ptw通过cache访存查找页表项之后，会重新填充tlb的entry，同时tlb切换回`ready`态，在这状态下，tlb将会成功实现tlb hit从而完成地址转换

#### PTW（Page Table Walker）的设计实现

ptw是一个用于实现页表访问的模块，当tlb出现缺失时，它会将tlb的重填请求解析为一系列的访存操作，然后通过内部状态切换+逐级访存最终实现tlb的重填

ptw的状态共有四个：`ready`, `request`, `wait1`, `wait2`

1. ptw的`ready`态说明此时它可以接受一个tlb重填请求，一旦接受这样的请求，就会切换状态到`request`，同时保存此时的虚拟地址中的一级页表号，二级页表号，页内偏移等等信息
2. `request`态开始访问第一级页表，一旦访存成功就保存当前的pte，然后切换到`wait1`态
3. `wait1`态等待一级访问，一旦出现成功访问的信号，就解析刚刚通过访问获得的pte，判断是否出现page fault，一旦出现page fault将重新退回到`ready`态，否则就将开始执行第二级页表的访问，并将状态切换到`wait2`态
4. `wait2`态等待二级访问，一旦出现成功访问的信号，就解析刚刚通过访问获得pte，判断是否出现page fault，出现page fault就退回到`ready`态，否则就将最终访存获得物理页号返回给上一级，返回`ready`态



## 模块设计-双核部分

### Cache

为提高妨存效率，避免两个核每次访存操作都访问总线，我们为System on Cat设计了cache系统。具体地，我们的cache使用了多路组相连的映射方式、全局的Round-Robin替换策略以及基于写更新和总线监听的一致性协议。我们将cache的路数、组数和每个cache块包含的数据长度都设计为Cache模块的参数以方便尝试不同的配置组合。

具体地，每个cache项有三种状态：

- Invalid：该项不对应任何数据
- Dirty：有数据与该项对应，且cache中的副本可能与总线上的不一致
- Clean：有数据与该项对应，且cache中的副本与总线上的一致

在只有一个核的情况下，这三种状态之间的转换很简单，我们无需赘述。下面我们将主要介绍我们为双核系统设计的cache一致性协议。

我们的cache一致性协议使用了总线监听的思路，也就是说每个cache都会将自己尝试进行的操作通知（广播）给另外一个cache，另外一个cache根据情况进行相应的操作并给出反馈。另外，我们使用了写更新的策略，即一个cache发现另外一个cache要对某个自己已缓存的数据块写操作时，它会自己cache块中的数据进行更新，而不是将其置为无效。这样，在任意时刻，每个数据块在其中一个或两个cache中均为Dirty或者均为Clean，在其余的cache中均为Invalid，不会有一个数据块在一个cache中为Dirty在另外一个cache中为Clean的情况；每个数据块在两个cache中的副本（如果都有的话）也一定是一致的。基于这一性质，当一个cache需要替换掉一个Dirty的数据块时，它若发现该数据块也存在于另一个cache中，便不用进行写回的操作。

下面列出了cache能够发出的广播消息类型：

- `FETCH`
- `MODIFY`
- `WRITE_BACK`

分别表示该cache即将进行的cache操作，即获取数据块、修改数据块和写回数据块。注意这些cache操作并不是与访存操作直接对应的，每一次访存操作可能包含零个或多个cache操作。当访存的读操作直接命中时，它将不会引起任何cache操作，而当访存的写操作需要替换并写回脏的数据块时，它会先后引发`WRITE_BACK`, `FETCH`和`WRITE_BACK`三个cache操作。广播中除了消息类型外，还有地址及数据等描述该操作的详细信息。当然，cache也可以不发出任何广播消息（或者说发出一条特殊的广播消息`NO_MESSAGE`）。

在收到广播消息时，cache需要立即进行回复（或者没有回复）。下面对每种广播消息给出对应的回复消息的类型和含义，以及针对回复的处理方式（假设cache A给cache B广播了消息，用`NO_MESSAGE`表示没有无信息）：

| 广播消息         | 回复消息            | 回复含义                                   | 回复处理                         |
| ------------ | --------------- | -------------------------------------- | ---------------------------- |
| `FETCH`      | `CLEAN_FOUND`   | A所需数据块在B中存在且为Clean态                    | A直接从B的回复中获得数据块内容并将其标记为Clean态 |
| `FETCH`      | `DIRTY_FOUND`   | A所需数据块在B中存在且为Dirty态                    | A直接从B的回复中获得数据块内容并将其标记为Dirty态 |
| `FETCH`      | `NO_MESSAGE`    | A所需数据块在B中没有                            | A开始从总线上获取数据                  |
| `FETCH`      | `STALL`         | A需要等待一周期                               | A保持当前状态，什么都不做                |
| `MODIFY`     | `NO_MESSAGE`    | B已根据A所给信息进行了处理，即如果B中存在相应数据副本，则对其进行相应修改 | A对自己的数据副本进行修改                |
| `MODIFY`     | `STALL`         | A需要等待一周期                               | A保持当前状态，什么都不做                |
| `WRITE_BACK` | `NO_WRITE_BACK` | A准备写回的数据块在B中存在                         | A跳过向总线写回数据的步骤                |
| `WRITE_BACK` | `NO_MESSAGE`    | A准备写回的数据块在B中不存在                        | A开始向总线写回数据                   |
| `WRITE_BACK` | `STALL`         | A需要等待一周期                               | A保持当前状态，什么都不做                |

注意到上面有一种比较特殊的回复消息类型`STALL`，它主要是为避免两个cache的操作产生冲突而设。下面我们首先讨论我们的cache系统对冲突的应对策略，然后再介绍cache回复`STALL`的具体情况。

我们将冲突分为两种类型：

1. 同步冲突：两个cache同时尝试进行cache操作
2. 异步冲突：一个cache在另一个cache执行操作期间尝试进行cache操作

两种类型的冲突均不是所有情况都会导致问题，但是我们简单起见决定不对这些情况进行细分，将避免冲突简化为保证cache操作的原子性，即在一个cache执行操作的时候另一个cache必须等待，当两个cache同时尝试进行操作时通过仲裁决定谁优先。

由上面的讨论，cache在收到广播消息后会判断自己是否正在进行操作，如果是则返回`STALL`，否则根据一个使用Round-Robin策略的cache仲裁器的输入信号来决定是否返回`STALL`。

由于时间和精力有限，我们的cache一致性协议的实现还有瑕疵，不能完全正常地工作。



### 原子指令

RISC-V的A扩展包含的原子指令有两种：LR/SC与AMO系列指令。这两种指令的原理并不相同，故需要分别实现。在rocket-chip系列中，原子指令是通过TileLink这个较强的总线协议实现的，而我们的总线没有写那么复杂，再考虑到IF段只会读出数据而不会修改，对于原子指令的过程与结果并无影响，故只在DMem元件中完成。

#### LR/SC

LR/SC(Load Reserved/Store Conditional)是实现同步机制的一种方式。LR指令从某个地址读出数据，SC指令试图向某个地址写入数据。如果对同一个地址的LR到SC之间，这个地址未被修改过，则判定SC成功，进行写入；若已被修改过，则SC失败，向目标寄存器rd里写入非零值表示操作失败。用这个机制，就可以完成对同一个地址的抢占。

看上去功能比较复杂，但文档对这一机制在软件上的使用做了简化处理，使得它只能用于实现规模较小的同步操作：规定一个核中生效的LR指令只能有一个；规定LR/SC整个处理流程不得超过16条指令。

实现比较简单。使用一个元件LRSCSynchronizer连接两个核的DMem元件，当某个核发生LR时，记录下其保留的地址；当两个核中的任意一个试图修改它时，则将其置为无效。SC时，检查保留地址是否与访问地址一致且有效即可。

一个特殊情况是，如果两个核同时执行同一段代码，即同时对同一个地址执行LR，之后又同时对同一个地址执行SC，则需要特别处理。首先显然不能让两核都SC成功，这样破坏了原子性；但如果判做两核都失败，由于两边运行的代码完全相同，两核又会以完全一致的时序重试，然后再次失败，一直循环下去，最终两个核都无法成功访问，彻底卡死。因此，需要进行仲裁，让其中某一个核成功写回，另一个核返回SC失败。

#### AMO

AMO系列包含AMOADD, AMOXOR, AMOOR, AMOAND, AMOMIN, AMOMAX, AMOMINU, AMOMAXU, AMOSWAP共9条指令。其基本功能是类似的：从某一地址读出数据，与另一操作数进行运算，再存回原地址。并且在这一过程中要保证硬件上的原子性。

为了实现这一功能，在DMem中添加了AMO这一元件。其功能是，当某一AMO系列指令到来时，由这一元件接管数据段的访问，其内部的一个状态机控制读出、运算、写回的全过程，直到操作结束，才解除阻塞状态，让流水线继续运转。

此外，为了完成其中运算这一步，还添加了一个微型的AMOALU，直接放在AMO元件内部。

AMO状态机包含sIDLE, sENTER, sLOAD, sLOAD_WAIT, sSTORE, sSTORE_WAIT六个状态，其功能及转化流程如下：

- sIDLE：空闲状态，流水线未阻塞，随时可以接受下一个原子操作请求；请求到来时进入sENTER.
- sENTER：等待AMOSynchronizer进入临界区，成功后进入sLOAD.
- sLOAD：向下层访存元件(SysBusConnector)发送读取请求，下一个周期直接进入sLOAD_WAIT.
- sLOAD_WAIT：等待，直至读取请求结束，SysBusConnector返回的阻塞信号为0时进入sSTORE.
- sSTORE：将读出的数据送入AMOALU进行运算，并发送写回请求。下一个周期直接进入sSTORE_WAIT.
- sSTORE_WAIT：等待，直至写回请求结束，即SysBusConnector返回的阻塞信号为0时回到sIDLE.

这里的逻辑本质上和PTW（见“MMU模块”一节）是一致的，故并不复杂。

需要注意的是sENTER一处。这一状态是为了避免两个核同时进行AMO操作而设置的，如果一个核在进行AMO操作的过程中另一个核也在访存，显然不能保证原子性。所以，我们加上AMOSynchronizer这个元件进行仲裁，当一个核执行AMO操作时，另一个核就被禁止进入sENTER态。

#### 同步细节

实现在DMem中的原子同步机制有很多的细节问题：

- 当一个核在执行AMO，而另一个核在试图写入同一个地址时，AMO的原子性将被破坏。因此，采用了最简单粗暴的方法：一个核执行AMO时会干脆停止另一个核访存，并且也会破坏其LR的预留有效性。
- 一个核执行LR之后，到SC之前，可能另一个核并没有执行新的访存操作，但有可能在LR之前另一个核已经发起了一次写入，但因MMU仍在读取页表而延缓了操作过程，导致最终生效在LR之后。对此，我们把发生LR时另一个核正在执行而未执行完的写入指令也考虑在内。
- 当一个核的LR和另一个核的Store同时发生且地址冲突时，由于究竟哪一条命令先完成是由总线决定的，上层无法得知，故我们认定为LR直接失效。

将以上这些同步逻辑全部加入之后，双核uCore+可以开机并进入sh了。

#### 疑虑

然而，当项目结束之后反思看来，以上这一套原子指令的设计是有缺陷的。正确的设计，还是必须让总线协议保证其原子性。因为我们的实现存在的重大问题是，地址比较均使用的是虚拟地址而非物理地址，这会在两核的页表不同时失效。

uCore+里只有自旋锁使用了原子操作，而这些自旋锁都只存在于内核态，即只存在于操作系统的数据段中，而操作系统数据部分的页映射都是完全相同的，这一切简单化的处理使得这个硬件bug并没有出现。但是如果要把这一套硬件用于其他程序，显然目前的实现还是不成立的。

并且，在MEM段实现原子指令也导致了同步过于复杂，如果实现在总线的话，总线负责阻塞其他核的难度和准确性要高上很多。最终双核仍然有一些bug，这可能还是同步机制里的细节问题没有得到解决的缘故。

## 软件项目

### Meownitor监控程序

为了在完成操作系统移植前为硬件系统提供一个相对较强的测例，我们参照16位THINPAD的做法，编写了一个相对操作系统更简单但对硬件中的各种关键机制有较全面的覆盖的监控程序。由于在编写监控程序的同时硬件的功能也在不断增强，对各个硬件特性的支持是以增量形式被添加到监控程序中的，通过设置编译选项能够十分方便地选择在监控程序中用到哪些硬件特性。这种无意中形成对的设计带了了几个好处：

1. 硬件和软件互为测例：硬件和软件各自都是以增量形式进行开发的，而在各个增量阶段硬件和软件能够互相作为测例。例如，在硬件更新后，可以令其运行旧的、已经经过测试的软件来检测其是否在改动的过程中破坏了已有的功能。
2. 模块化，方便问题的准确定位：可以通过控制各个功能的启用状态来准确定位问题所处的模块。

#### 核心

基本的监控程序核心在功能层面上基本上是对16位THINPAD监控程序的复刻。它通过串口进行输入和输出，支持如下几种指令：

- J: 跳转到指定的地址执行
- R: 查看上次执行结束时各个寄存器的值
- E: 编辑指定地址的数据
- I: 以汇编形式编辑指定地址的指令
- V: 查看指定地址的数据
- D: 以汇编形式查看指定地址的指令

与16位THINPAD监控程序不同的一点是，我们的监控程序不依赖于特殊的客户端，也就是说，所有的功能，包括汇编、反汇编、指令的解析等都是在监控程序中实现、在System on Cat上运行的。这会一定程度上增加监控程序的复杂性，因而我们使用C语言进行监控程序的实现，而不像16位THINPAD监控程序那样直接由汇编语言实现。

#### 时钟中断和异常扩展

时钟中断和异常扩展主要包含以下改动：

- 加入了对用户程序（即J跳转后到返回前执行的代码）的计时功能。
- 用户程序的执行加入了时间限制，在超过实现后会被监控程序抢占执行。这个时间限制可以通过T指令进行设置。
- 加入对异常（除environment call）的处理。包括非法指令、指令未对齐、数据读取未对齐和数据写入未对齐。

#### 外部中断扩展

外部中断扩展使监控程序能够以中断的形式响应并处理外设发出的请求。具体而言，它主要包括了如下改动：

- 串口的读取通过中断完成
- 加入软件实现的串口输入缓冲区，在终端中读取到的串口数据均放置到该缓冲区中

#### 系统调用扩展

系统调用扩展使用户程序能够通过系统调用机制（在RISC-V规范中被称为environment call）来获取监控程序的服务。具体地，在系统调用中，监控程序以`a1`寄存器存放调用号，`a2, a3, ...`存放参数，`a0`存放返回值。下表列出了监控程序支持的所有系统调用：

| 调用号  | 参数        | 返回值       | 说明             |
| ---- | --------- | --------- | -------------- |
| 0    | 无         | 无         | 退出用户程序，返回到监控程序 |
| 1    | 一个字节的字符数据 | 无         | 向串口输出一个字符      |
| 2    | 无         | 一个字节的字符数据 | 从串口读入一个字符      |

### uCat操作系统

uCat操作系统基于上学期操作系统课程设计中的uCore+ RISC-V 64G (含LKM和SMP支持)的成果。主要的移植工作包括：

- 为System on Cat实现bootloader
- 从64G移植到32I
- 地址空间布局调整
- 更改访问外设的接口

下面我将对其中比较关键的部分进行更加详细的介绍。

#### BootedCat

上学期的操作系统课程设计中使用bbl进行系统的初始化和引导，但是经过一些研究之后我们认为它在我们的系统中并不十分适用，主要原因如下：

1. bbl适用于操作系统运行在S态的系统，而在我们的设计中，操作系统直接运行在M态，用户程序运行在U态（这种设计更加简单）
2. bbl的实现过于复杂，诸如device tree的初始化等很多功能我们都用不上
3. bbl不一定能够适配我们的外设接口

出于上述考虑，我们为自己的系统实现了叫做BootedCat的bootloader。它做的工作十分简单：

1. 从Flash中预定的位置读取ELF header。我们规定Flash中4096这个地址开始存放要加载的程序的可执行ELF镜像。
2. 对ELF header进行sanity check。如果sanity check成功则继续执行，否则输出错误信息并进入死循环。
3. 配置并启用初始页表。由于ELF镜像中的各段可能位于较高的地址，超出了RAM的地址空间，在bootloader中我们就需要使用虚实地址转换。具体地，我们将0xC0000000开始的8MB虚地址空间映射到0x0开始的8MB，将0x0开始的一页（包含bootloader本身）做恒等映射，包含外设物理地址（除了ROM，因为ROM在之后不会再被用到）的那一页也做恒等映射。在经过这个映射下，操作系统可通过加减0xC0000000来完成虚实地址的转换。
4. 读取ELF中的各个program headers，将各段的数据加载到指定的地址。
5. 跳转到ELF镜像中指定的入口地址，将控制权交给操作系统。

#### uCore+ RISC-V 64G到32I移植

这一步移植工作比较繁琐，继续细分的话可以分为两个方面：64位到32位的移植以及G到I的移植。

64位到32位移植的工作主要包括：

- 很多地方的`uint64_t`要替换成`uint32_t`。
- 很多64位常量需要改成32位的。这既包括一些比较直接的（例如CSR设置相关的），也包括一些需要设计的（例如地址空间布局相关的）。
- 页表模式改为Sv32。由于uCore+的页表实现非常灵活，这一部分的工作基本上只需要修改一个描述页表机制的头文件里定义的各个常量就可以完成了。

从G移植到I的工作主要包括：

- 去掉A指令集的使用：将`atomic.h`中原子操作的实现替换成普通的访存操作。
- 去掉M指令集的使用：实现`__mulsi3`等一系列整数乘除和取模运算代替M中的指令，在编译时设置`-march=rv32i`可以使编译器自动将这些运算处理处理成函数调用。
- 去掉FD指令集的使用：原来的uCore+系统中多核进程调度的部分使用了少量浮点运算。由于我们的系统不支持多核（非挑战部分），我们可以直接将这些涉及浮点运算的代码删去。

#### 地址空间布局调整

uCat的虚拟地址空间布局如下表所示：

| 地址范围                  | 描述                       |
| --------------------- | ------------------------ |
| 0xC0000000-0xC07FFFFF | Remapped Physical Memory |
| 0x800000-0xBFFFFFFF   | User Program & Heap      |
| 0x200000-0x7FFFFF     | User STAB Data           |

#### 其他

在移植过程中我们发现一个RISC-V开发需要注意的地方：编译器/链接器生成的ELF中有时包含一个叫做`__global_pointer`的符号，这个符号与重定位相关，初始化代码将该符号的值赋给`gp`寄存器才能确保代码主体部分能够正确执行。

LKM部分的移植比较简单，主要是修改重定位类型及处理编译内核和模块时一些类型定义的差异：

- `R_RISCV_64`类型的重定位要替换为`R_RISCV_32`类型的重定位。
- 在内核中`bool`类型的定义和模块中的不一致，导致两者之间不能正确地进行数据的交换。把它们的定义统一一下即可解决这个问题。

经过移植后，我们就能够在uCat中动态加载并使用SFATfs模块了。

### 软件/固件的SMP支持

为了支持SMP，软件和固件方面也需要做出一些调整，这些调整的必要性大部分都是由于在系统初始化、设置RAM等共享资源状态的阶段需要有分工和同步的机制。

以MasterCat为例，两个核同时将Flash中的内容加载到RAM中无疑会导致效率降低。因此，我们只让其中一个核进行这个初始化操作，另一个核等待初始化操作完成后再继续执行。在BootedCat和uCat中，我们也采用了同样的机制。

在uCat中，由于需要使用原子指令实现双核间的同步互斥，我们还需要将之前移植过程中改掉的原子操作实现改回来，将之前删掉的一些自旋锁重新加上。另外就是调度那块的代码之前由于涉及到了浮点运算被删掉了，现在也要加回来并且想个办法替代掉浮点运算。

最后，我们在双核系统上完成了uCat的引导并顺利启动了`sh`程序，但在系统上进行后续的操作会导致一些不明原因的问题。

### Decaf编译器Decat

在已提供的Decaf语言mips编译器的基础上，我们实现了Decaf语言的Risc-V编译器（DeCat），并成功将使用Decaf高级语言编写的应用程序交叉编译为.S文件在uCore操作系统的调度下在SystemOnCat上运行

#### Decaf 编译器后端简介

Decaf编译器的前端工作到生成三地址码为止，接下来的后端工作主要有以下几个部分：

1. 将原有的三地址码划分为基本块，进行简单的数据流分析，构造数据流图，并进行简单的代码优化
2. 进行指令选择，将原有的三地址码的指令替换为对应平台支持的指令
3. 进行寄存器分配，将原有三地址码中无限的寄存器替换为对应平台支持的有限量的寄存器
4. 根据对应平台上的操作系统，添加相对应的系统调用，并将系统调用库函数和生成的.S汇编代码链接生成可执行文件

接下来我们分别介绍在基于Risc-V32的SystemOnCat计算机系统中，上述四个部分的具体实现

#### 划分基本块和构建数据流图

基本块划分是代码分析的第一步，SystemOnCat系统保留了原有mips编译器系统的实现方法，以出口语句（包括Return, Beqz, Bnez, Branch）为标志将原有的三地址码划分为一系列基本块，每一个基本块有它的后继基本块，同时还记录了每一个基本块中的活跃变量信息（如Def集合，LiveUse集合，LiveIn集合，LiveOut集合等），基于这些活跃变量信息，我们可以得到离开基本块时必须保存的寄存器集合

划分好基本块以后，可以在此基础上构建流图，流图体现了基本块之间的信息，对于一些不可达的基本块可以进行删除，从而实现简单的代码优化

#### 指令选择

由于原有的Decaf编译器是面向mips平台的，因此我们需要将相应的mips指令都替换为Risc-V指令，这些Risc-V指令需要保证和三地址码指令等价

基本上大部分三地址码指令都能够在Risc-V指令集中找到和它对应的指令，只有各别指令需要做一些替换，以下是部分需要用多条Risc-V指令来实现的三地址码指令

1. GEQ指令，要求在针对两个操作数比较时，A大于等于B则返回1，否则返回0，但是在Risc-V指令中只有SLT指令，即A小于B返回1，否则返回0，所以这条指令可以看成是和SLT意义恰好相反的指令，因此可以使用SLT和异或0实现
2. EQU指令，要求在针对两个操作数比较时，A等于B则返回1，否则返回0，可以使用Risc-V指令中的减法指令和判断是否为0的指令来组合实现
3. GTR指令，和SLT指令的意义恰好相反，因此只需要交换A，B操作数即可

此外由于SystemOnCat平台目前只支持Risc-V32IA系列指令，并没有实现M类型指令（包括各类乘除法和取余等），因此需要将乘除法和取余都改为函数调用的形式，使用软件来实现相应的逻辑，但是函数的调用有可能会破坏基本快的划分和数据流图的构建，需要提早完成这一替换过程 ———— 我们修改了decaf编译器的前端，在生成三地址码时就实现替换，保证对应代码中没有MUL, DIV, MOD等指令，将他们统统修改为IntrinsicCall的形式

#### 寄存器分配

由于原有的三地址码可以使用的寄存器是无穷多的，因此需要使用一定的分配算法将无穷个寄存器替换为有限多个，由于该分配算法和平台无关，因此我们保留了原有的mips编译器的相关实现

但是由于mips寄存器和Risc-V寄存器的名称和使用规范是不同的，因此需要针对原有的寄存器名称进行修改，同时需要指定哪些寄存器属于通用寄存器

#### 系统调用

为了能够获得平台的服务，每一个Risc-V应用程序需要能够通过系统/库函数调用来实现诸如输入，输出，内存分配，退出，乘法，除法，取余等操作，这些调用是和平台密切相关的，下面介绍一个简单的系统调用（如输出一个字符串）的实现流程：

1. 在.S文件中的代码段的起始位置会有一个简单的调用入口 _PrintString，其中实现了参数的传递，针对真正的库函数（_catlib__Printstring)的调用，以及调用结束之后的返回等一系列操作，在整个汇编代码中每一个需要输出字符串的位置就调用 call _Printstring 即可
2. 真实的库函数是使用c编写的catlib.c文件，这个文件使用Risc-V自带的工具链编生成对应的.o文件，由于我们的编译器和Risc-V自带工具链的编译器的调用规范保持了较好的一致性，因此可以直接将我们自己的编译生成的.S经过汇编器生成的.o与库函数的.o链接生成可执行文件
3. 在catlib.c文件中的_catlib__PrintString函数调用了uCore中提供的用户库函数cprintf，该函数将通过系统调用的形式要求uCore提供相应的服务

关于其他函数调用功能，如乘法，除法，取余等的实现过程类似，只不过catlib.c中调用的用户库函数直接在用户态就实现了相应的功能，而不需要通过系统调用的形式切换到内核态再实现相应的功能了

#### 实验结果

我们最终成功在自己的硬件上运行了uCore操作系统，并在操作系统的调度下运行了blackjack.decaf, math.decaf, fabonacci.decaf等应用程序

交叉编译器的具体使用方法参考DeCat文件夹下的Readme文件