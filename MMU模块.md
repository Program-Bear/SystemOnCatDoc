## MMU
MMU（Memory Management Unit）模块能够完成虚实地址之间的转换，通过虚存管理机制，实现了不同进程之间的地址隔离

### MMU概览及顶层模块MMUWrapper的设计
在SystemOnCat中MMU主要由TLB，PTW两个模块组成，它向上和综合流水线的两个访存段的Connector部件相连，这个Connector部件能够协调来自IF和MEM两个段的访存请求，同时还需要和另一个核的Connector交互，从而方便原子指令的实现；它向下和Cache部件相连，所有的访存通过Cache，然后Cache再和实际挂载在总线之上的Ram交互，Cache及其一致性的实现参见Cache设计文档

MMU的顶层封装模块称为MMUWrapper，这个模块将tlb和ptw实例化，内部有一个状态机，能够支持地址转换和实际访存两种状态的切换：

1. 状态1:进行地址转换，查找tlb需要一个周期，如果tlb发现自己缺失，它会去ptw中进行页表访存，此时将阻塞住上层mmu的访存过程，mmu保持在状态1，如果转换顺利完成进入状态2开始进行实际的访存，否则出现缺页异常通知给更上层的流水线

2. 状态2:一旦地址转换顺利完成，就通过拿到的地址进行内存的访问

### TLB 的设计实现
TLB是页表的缓存，在我们的设计中，TLB内含一个小型的状态机，共包含两个状态（ready, request）：

1. ready态表明当前的tlb能够接受一个新的地址转换请求，在ready态的tlb一旦接收到一个tlb请求，就会首先去查看tlb entry，看这个请求中包含的物理地址是否已经在tlb表中，查找的过程为直接映射，物理地址的低12位作为页内偏移，中间的5位作为index，剩下的部分连同asid作为tlb表的tag，每当找到一个对应的tlb entry时，就会对比它的tag是否和当前缓存的tag一致，如果一致就说明tlb hit，否则说明tlb misstlb hit不会导致状态的切换，tlb miss将使得tlb内的状态机切换到request态

2. request状态下，tlb会向更底层的ptw发出tlb refill请求，这一请求将被ptw解析为一系列的访存操作，ptw通过cache访存查找页表项之后，会重新填充tlb的entry，同时tlb切换回ready态，在这状态下，tlb将会成功实现tlb hit从而完成地址转换

### PTW（Page Table Walker）的设计实现
ptw是一个用于实现页表访问的模块，当tlb出现缺失时，它会将tlb的重填请求解析为一系列的访存操作，然后通过内部状态切换+逐级访存最终实现tlb的重填

ptw的状态共有四个：ready, request, wait1, wait2

1. ptw的ready态说明此时它可以接受一个tlb重填请求，一旦接受这样的请求，就会切换状态到request，同时保存此时的虚拟地址中的一级页表号，二级页表号，页内偏移等等信息

2. request态开始访问第一级页表，一旦访存成功就保存当前的pte，然后切换到wait1态

3. wait1态等待一级访问，一旦出现成功访问的信号，就解析刚刚通过访问获得的pte，判断是否出现page fault，一旦出现page fault将重新退回到ready态，否则就将开始执行第二级页表的访问，并将状态切换到wait2态

4. wait2态等待二级访问，一旦出现成功访问的信号，就解析刚刚通过访问获得pte，判断是否出现page fault，出现page fault就退回到ready态，否则就将最终访存获得物理页号返回给上一级，返回ready态

