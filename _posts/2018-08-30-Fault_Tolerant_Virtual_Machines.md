---
layout:     post
title:      "The Design of a Practical System for Fault-Tolerant Virtual Machines"
subtitle:   "paper reading"
date:       2018-08-30 19:02::00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - 技术随想
---

## 概述
<br>这篇论文是VMware公司对虚拟机失败容忍处理方面的介绍性论文。它通过对虚拟机实现一个（或多个）备份进行失败容忍，这套系统在VMware自己的软件上运用，但在发表论文时只能在单处理器上运行，针对design的细节论文给予比较细致的解答。<br/>
<br>实现失败容忍服务手段主要有两种：第一种将主服务器上改变的状态全部发送到backup的机器上（包括CPU、memory、I/O device），这对带宽要求非常高。第二种是将服务器看作确定性状态机，将它们从相同的初始状态开始，经过相同顺序的请求来保证它们同步，但是有些操作时不确定的，比如并发写状态和中断请求，所以必须对这些operation和event进行全部捕捉和排序，这种操作相比上一种需要传输的数据就少了很多，而且虚拟机能够更好的实现第二种情况，因为虚拟机有hypervisor（VMM）能够准确控制虚拟机的一切操作（中断时间，时钟读取，请求排序），可以将虚拟机看作一台完美的确定性状态机。<br/>
<br>论文使用determine replay技术进行状态机状态迁移，简单点说就是primary将自己的operation、event打成log传输给backup，backup通过重放这些operation，将自己的状态机达到和primary相同的状态。当primary失败的时候，backup能够直接接手，从而对client是透明的，值得注意的是这篇论文针对的失败是fail-stop的失败，也就是failure只要出现，服务器直接宕掉，不负责处理会导致导致server状态改变，但是服务器照常运行的错误。<br/>

## 基础设计
<br>论文为了清晰、简单说明，backup选择一台作为解释。primary和backup运行在两台服务器上的虚拟机上，上面各自运行os和app，但是在基础设计里磁盘采用的共享磁盘，这样有利于失败迁移（不用迁移大头的磁盘空间，之后会讨论分离的disk）。hypervisor维持一个logging cahnnel（就是一个大的bufffer，primary尽快将自己的log从自己的log buffer刷进logging channel，backup尽快将logging channel的数据取到自己的buffer里）来将log在primary和backup之间传递。这套系统大头的传输流量来自于input的输入（disk和network），这套系统保证backup和primary保持相同的状态，但是backup的output直接被丢弃。<br/>
<br>determine replay实现关键是将必要的操作按照order在所有服务器上重现，从而保持相同的状态。主要的操作包括input（disk，network，mouse and keyboard。论文将一般的命令看成input，毕竟需要从其他地方输入），不确定event（虚拟中断），不确定operation（读取处理器时钟）。所以determine replay面临三个挑战：1.正确捕获全部input和不确定事物2.成功apply input和不确定事物到backup上3.不会大幅损失性能。此外，X86处理器上很多复杂操作有side effects，系统需要捕获这些不确定性操作，replay them。<br/>
<br>VMware捕捉input和不确定execution打进log里，不确定性operation将充足的信息写进log里，不确定性event（类似中断完成操作），primary产生的确定的指令（instruction）打进log能够让backup在正确的时间产生相同的event，在replay时候，这种event会被正确插入instruction stream。以下例子来自MIT课件<br/>
>Example: timer interrupts
Goal: primary and backup should see interrupt at exactly the same point in   execution  
    i.e. between the same pair of executed instructions  
  Primary:  
    FT fields the timer interrupt  
    FT reads instruction number from CPU  
    FT sends "timer interrupt at instruction X" on logging channel  
    FT delivers interrupt to primary, and resumes it  
    (this relies on special support from CPU to count instructions, interrupt   after X)  
  Backup:  
    ignores its own timer hardware  
    FT sees log entry *before* backup gets to instruction X  
    FT tells CPU to interrupt at instruction X  
    FT mimics a timer interrupt, resumes backup  

>Example: disk/network input  
  Primary and backup *both* ask h/w to read  
    FT intercepts, ignores on backup, gives to real h/w on primary  
  Primary:  
    FT tells the h/w to DMA data into FT's private "bounce buffer"  
    At some point h/w does DMA, then interrupts  
    FT gets the interrupt  
    FT pauses the primary  
    FT copies the bounce buffer into the primary's memory  
    FT simulates an interrupt to primary, resumes it  
    FT sends the data and the instruction # to the backup  
  Backup:  
    FT gets data and instruction # from log stream  
    FT tells CPU to interrupt at instruction X  
    FT copies the data during interrupt  

<br>论文采用FT protocol来保证失败容忍。整个系统最重要的要求就是如果backupVM在primary失败后上线后，backup VM能够以和primary已经输出到外界的output一致的操作执行。以下内容引用自MIT 6.824课件内容<br/>
>Why the Output Rule?  
  Suppose there was no Output Rule.
  The primary emits output immediately.
  Suppose the primary has seen inputs I1 I2 I3, then emits output.
  The backup has received I1 and I2 on the log.
  The primary crashes and the packet for I3 is lost by the network.
  Now the backup will go live without having processed I3.
    But some client has seen output reflecting the primary having executed I3.
    So that client may see anomalous state if it talks to the service again.
  So: the primary doesn't emit output until it knows that the backup
    has seen all inputs up to that output.

<br>为了实现这一要求，协议规定backup必须接收到output operation之前所有的log entries（我不明白，难道output operation 之后的log entry就可以丢弃吗？这不会造成client认知上的不一致吗？这时候primary buffer的log没传到logging buffer就全部丢失了，是因为这些failure不属于fail-stop failure，就不负责处理？），backup上线时机必须replay到output operation这一点上，否则直接上线的话，会导致在output之前上线（不用primary给予中断命令），就出现各种不确定性事件（例如时间中断）。论文给出的方法是在primary执行output operation之前需要backup回复一条特殊log表明收到operation log（自然之前的log全部收到）。注意虽然需要推迟primary output的时间，不代表需要primary停止运行（这里的解释我不太明白，如果继续执行，会执行什么呢？难道不会对需要output的数据产生影响吗？）。当这个规则使用后，表明backup上线的时候会和primary last output的状态一致（表明会抛弃那些output之后没来得及传的log）。对了，这种做法可能会导致output两次（primary在执行完output后crash，但此时backup还没执行到output），就会执行两次output，这时候保证看起来只有一次output的措施是
>  OK for TCP, since receivers ignore duplicate sequence numbers.  
   OK for writes to shared disk, since backup will write same data to same block #.

<br/>
<br>失败检测和恢复。当backup VM失败的时候，primary VM开始执行normal mode（不用发送log entry到logging channel）。当primary VM失效时，backup VM成为primary上线，但是需要一点时间来让它消费log buffer的log entries，在这之后，backup VM会从replay mode切换到normal mode，开始向外界执行output操作，当然为了识别谁是primary，需要硬件上支持通过MAC address识别。FT识别失败通过UDP心跳机制定期ping每个服务器和监控logging traffic（因为有timer interrupt，所以会定期发送log）。但是如果是网络连接中断，这会导致split-brain问题，两个VM都认为自己primary然后上线，论文通过对共享存储空间进行atomic test-and-set operation，只有能够成功的VM，才能够作为primary上线，剩下的那个需要自我降格为backup，不能的VM会一直尝试。注意如果shared storage不能够访问的话，那VM也自然都失效，因为shared disk也在这上面。另外一点，如果一个VM失败但是另个上线，上线的会为了恢复冗余性重启一个backup VM。<br/>

## FT实现细节
<br>启动和重启FT VMs。需求是随时启动一个和primary一样状态的backup，论文只是介绍VMware的VMotion操作（用来VM迁移）可以改动一下用来复制primary VM，它会导致source VM变成logging mode，设置一个logging channel，然后被复制的VM变成replay mode。关于backup VM位置选择：因为所有的VM都运行在集群的服务器上，它们能够共享存储空间，所以选取任意一台服务器。VMware vSphere实现集群服务来维护管理和资源信息，集群服务会确定最佳backup VM位置，然后执行备份操作。<br/>
<br>logging channel管理。logging的管理分为hypervisor维护的在primary和backup的log buffer，通过logging channel连接。primary和backup都尽可能快得将log entries写入和读出。主要问题在于primary将buffer写满了，这样primary不得不完全停止，backup的buffer空不是个问题，它停止就可以了，毕竟client不会直接于backup联系。导致primary buffer满的主要原因是因为backup消费log太慢，原因一在于其实服务器配置相似，服务器可能会运行太多VM。另一点不想backup落后primary太多的原因在于backup接替primary主要花费的时间就是失败检测时间+replay这些log的时间。所以，论文利用额外机制探测primary和backup之间相差的lag，如果一直太大，逐渐降低primary执行速度，相反如果lag逐渐缩小，就逐渐提高primary速度。<br/>
<br>关于VM operation问题。一些特殊控制命令也需要打进log里（关机，调整资源）。一般来说，operation由primary发起。唯一primary和backup可以独立进行的操作是VMotion。关于这套系统的VMotion需要断开和另一方的连接，然后重连到目的地。对于VMotion需要所有I/O操作停止，对于primary很简单，等物理I/O结束就行。但是对backup有问题，因为backup要准确操作点执行primary操作，所以当backup要执行VMotion时候，backup发送请求给primary，要求它停止所有I/O操作。(有点小问题，可能是因为确定到Instruction的数值)<br/>
<br>Disk I/O。因为对磁盘的操作是不堵塞的，所以可以并行，还有DMA操作同时对同一块disk进行操作。另外，disk I/O使用DMA技术，所以disk read和VM read会在同一块memory冲突，这两种情况都会导致不确定性。第一个问题是检测这种data race，然后按顺序执行。第二种问题论文给出方案是bounce buffer，当disk read或者写进disk时候，先传到bounce buffer，等到完成之后，在读到VM能够接触的内存或者刷进disk里。保证的是在相同的操作点看到memory的数据是一样的。
>FT avoids this problem by not copying into guest memory while the
primary or backup is executing. FT first copies the network packet or
disk block into a private "bounce buffer" that the primary cannot
access. When this first copy completes, the FT hypervisor interrupts
the primary so that it is not executing. FT records the point at which
it interrupted the primary (as with any interrupt). Then FT copies the
bounce buffer into the primary's memory, and after that allows the
primary to continue executing. FT sends the data to the backup on the
log channel. The backup's FT interrupts the backup at the same
instruction as the primary was interrupted, copies the data into the
backup's memory while the backup is into executing, and then resumes
the backup.  
The effect is that the network packet or disk block appears at exactly
the same time in the primary and backup, so that no matter when they
read the memory, both see the same data.

最后一点是primary失败，那些diskI/O怎么处理，因为backup不知道是否成功，论文给出的方案是backup全部重新执行。（因为是幂等的）<br/>
<br>网络I/O。论文抛弃之前采用的直接异步更新网络I/O设备状态，因为这会带来更多的不确定性。论文采用guest给hypervisor trap的方式使得VM log这些update（去除不确定性）。为了缓解这个问题，需要尽可能少的trap（几个包并在一起trap），hypervisor也可以打包几个在给VM一个中断。第二个优化就是在primary发送output entry和接受回执时候不切换上下文，尽量加速这个过程，让网络包尽快发出去。<br/>

## 替换设计
<br>基础设计一直是shared disk，其实也可以用分离的磁盘。在shared disk时候，backup不能写（disk算是外部output），而且失败发生时，disk是可以直接用的。分离磁盘，backup需要直接写磁盘了，但是由于disk属于backup内部状态，所以不用遵循output rule了，两份磁盘必须显示同步当FT被启用时，而且失败时也需要重新同步，VMotion需要传递disk state。关于split-brain问题，只能求助第三方服务器，谁节点多谁是primary。<br/>
<br>在原始设计里，backup从来不会从disk里读取数据（shared and non-shared），都是primary通过logging channel传给backup。所以一种设计时backup直接读取磁盘，这可以大幅减少logging channel的流量，但是会面对多个巧妙的问题。第一，这会减慢backup执行速度，因为当执行到primary完成的执行点时，backup必须等到自己的read完成，才能执行下一条指令。2.需要执行一些其它操作来应对失败，如果backup失败、primary成功，backup需要一直重复直到成功，如果primary失败、backup成功，这时候primary里的目标位置内存需要通过logging channel传输。3.针对共享磁盘，防止出现primary读之后迅速写，导致backup读出现错误，需要更多操作保证。<br/>
