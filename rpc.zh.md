
# {{Paj.Toe}}

正如在第1章中所讨论的,应用程序使用的一种常见通信模式是请求/应答范例,也称为*消息事务*客户机向服务器发送请求消息,服务器使用应答消息进行响应,客户机阻塞 (暂停执行) 以等待应答. [图1](#rpc-timeline)说明在这样的消息事务中客户端和服务器之间的基本交互. 

<figure class="line">
	<a id="rpc-timeline"></a>
	<img src="figures/f05-13-9780123850591.png" width="300px"/>
	<figcaption>Timeline for RPC.</figcaption>
</figure>

支持请求/应答范例的传输协议远不止是沿着一个方向前进的UDP消息,然后是沿着另一个方向前进的UDP消息. 它需要处理正确地识别远程主机上的进程,并将请求与响应关联起来. 它还可能需要克服本章开头问题说明中概述的底层网络的一些或全部限制. 虽然TCP通过提供可靠的字节流服务来克服这些限制,但是它与请求/应答范式很不匹配,或者麻烦地建立TCP连接,只是为了交换一对消息,好像是过多的. 本节描述了第三种类型的传输协议,称为*远程过程调用* (RPC) ,它更紧密地匹配请求/应答消息交换中涉及的应用程序的需求. 

## RPC基本原理

RPC实际上不仅仅是一个协议,它是构建分布式系统的流行机制. RPC是流行的,因为它是基于本地过程调用的语义. 应用程序在不考虑它是本地的还是远程的情况下调用一个过程并阻塞直到调用返回. 应用程序开发人员在很大程度上不知道程序是本地的还是远程的,大大简化了他的任务. 当被调用的程序实际上是面向对象语言中的远程对象的方法时,RPC被称为*远程方法调用* (RMI) . 虽然RPC概念很简单,但有两个主要问题使得它比本地过程调用更复杂: 

-   调用进程和被调用进程之间的网络具有比计算机底板复杂得多的特性. 例如,它很可能限制消息大小,并有丢失和重新排序消息的倾向. 

-   运行调用和被调用进程的计算机可能有显著不同的体系结构和数据表示格式. 

因此,一个完整的RPC机制实际上包括两个主要部分: 

1.  一种协议,用于管理客户端和服务器进程之间发送的消息,并处理底层网络的潜在不良属性

2.  编程语言和编译器支持将参数打包到客户端机器上的请求消息中,然后将此消息转换回服务器机器上的参数,同样地,还支持返回值 (RPC机制的这一部分通常称为*存根编译器*) 

[图2](#rpc-stub)示意性地描述了当客户端调用远程过程时会发生什么. 首先,客户端调用过程的本地存根,传递过程所需的参数. 该存根通过将参数转换为请求消息,然后调用RPC协议将请求消息发送到服务器机器来隐藏过程是远程的事实. 在服务器上,RPC协议将请求消息传递给服务器存根 (有时称为*骨骼*) ,它将其转换为过程的参数,然后调用本地过程. 在服务器过程完成之后,它将答案返回给服务器存根,服务器存根将此返回值打包在应答消息中,并将该应答消息交给RPC协议以传输回客户端. 客户机上的RPC协议将这个消息传递给客户机存根,客户机存根将其转换为返回给客户机程序的返回值. 

<figure class="line">
	<a id="rpc-stub"></a>
	<img src="figures/f05-14-9780123850591.png" width="500px"/>
	<figcaption>Complete RPC mechanism.</figcaption>
</figure>

本节仅考虑RPC机制的协议相关方面. 也就是说,它忽略存根,而是将重点放在在客户端和服务器之间传输消息的RPC协议 (有时称为请求/应答协议) 上. 将参数转换成消息和*反之亦然*覆盖在别处. 

术语*RPC*指的是一种类型的协议,而不是像TCP这样的特定标准,所以特定的RPC协议执行的功能各不相同. 而且,不像TCP,它是占主导地位的可靠字节流协议,没有一个占主导地位的RPC协议. 因此,在本节中,我们将谈论更多的替代设计选择比以前. 

### RPC中的标识符

任何RPC协议必须执行的两个功能是: 

-   提供一个名称空间,用于唯一标识要调用的过程. 

-   将每个回复消息匹配到相应的请求消息. 

第一个问题与识别网络中的节点的问题有一些相似之处,这一点我们在前面几章 (例如,IP地址) 中已经看到. 标识事物时的设计选择之一是使这个名称空间平坦还是分层. 平面名称空间将简单地为每个过程分配唯一的ㄡ非结构化的标识符 (例如,整数) ,并且这个数字将在RPC请求消息的单个字段中携带. 这将需要某种类型的中心协调避免向两个不同的过程分配相同的程序编号. 或者,协议可以实现一个分层名称空间,类似于用于文件路径名的名称空间,它只需要文件的"basename"在其目录中是唯一的. 这种方法潜在地简化了确保过程名唯一性的工作. RPC的分层名称空间可以通过以请求消息格式定义一组字段来实现,每个级别的命名 (例如,二级或三级分层名称空间) 都有一个字段. 

将应答消息与对应请求匹配的关键是使用消息ID字段唯一地标识请求-应答对. 答复消息将其消息ID字段设置为与请求消息相同的值. 当客户端RPC模块接收到应答时,它使用消息ID搜索相应的未完成请求. 为了使RPC事务看起来像对调用方的本地过程调用,调用方被阻塞 (例如,通过使用信号量) ,直到接收到应答消息. 当接收到应答时,基于应答中的请求号码来识别被阻塞的调用方,从应答中获得远程过程的返回值,并且解除阻塞,以便调用方能够以该返回值返回. 

RPC中的一个经常性的挑战是处理意外的响应,并且我们使用消息ID看到这一点. 例如,考虑以下病理 (但现实) 的情况. 客户端机器发送消息ID为$0$的请求消息,然后崩溃并重新引导,然后发送不相关的请求消息,消息ID也为$0$. 服务器可能没有意识到客户端崩溃并重新启动,并且在看到消息ID为$0$的请求消息时,确认该消息并将其作为副本丢弃. 客户端永远不会收到请求的响应. 

消除这个问题的一种方法是使用*启动ID*. 机器的引导ID是每次机器重新启动时递增的数字;这个数字是从非易失性存储器 (例如,磁盘或闪存驱动器) 读取的,递增的,并在机器的启动过程中写入存储设备. 然后将该数字放入主机发送的每个消息中. 如果用旧消息ID接收新消息,但新的启动ID被识别为新消息. 实际上,消息ID和引导ID组合以形成每个事务的唯一ID. 

### 克服网络限制

RPC协议通常执行额外的功能来处理网络不是完美信道的事实. 两个这样的功能是: 

-   提供可靠的消息传递

-   通过拆分和重组支持大消息大小

RPC协议可能实现可靠性,因为底层协议 (例如,UDP/IP) 不提供可靠性,或者可能更快或更高效地从故障中恢复,否则最终将由底层协议修复. 类似于TCP,RPC协议可以使用确认和超时来实现可靠性. 基本算法是简单的,如给出的时间线所示. [图3](#chan-timeline1). 客户端发送请求消息,服务器确认该请求消息. 然后,在执行该过程之后,服务器发送回复消息,并且客户端确认答复. 

<figure class="line">
	<a id="chan-timeline1"></a>
	<img src="figures/f05-15-9780123850591.png" width="200px"/>
	<figcaption>Simple timeline for a reliable RPC protocol.</figcaption>
</figure>

携带数据的消息 (请求消息或应答消息) 或发送来确认该消息的ACK可能在网络中丢失. 为了解释这种可能性,客户端和服务器都保存它们发送的每个消息的副本,直到它的ACK已经到达. 每一方还设置一个重传计时器,并在该计时器到期时重新发送消息. 双方重置该计时器,并在放弃和释放消息之前再试几次商定的次数. 

如果RPC客户端接收到应答消息,显然服务器必须接收到相应的请求消息. 因此,答复消息本身是一个*隐式确认*而且,任何额外的来自服务器的确认都不是逻辑上必要的. 类似地,请求消息可以隐式地确认前面的应答消息ℴℴ假设协议使请求-应答事务顺序进行,以便一个事务必须在下一个事务开始之前完成. 不幸的是,这种顺序性将严重限制RPC性能. 

摆脱这种困境的一个方法是RPC协议实现一个*通道*抽象化. 在给定通道内,请求/应答事务是顺序的-在任何给定时间只能在给定通道上激活一个事务-但是可以有多个通道. 每个消息都包含一个通道ID字段,用于指示消息属于哪个信道. 如果给定通道中的请求消息尚未被确认,则该通道中的请求消息将隐式地确认该通道中的先前应答. 如果一个应用程序想要同时在它们之间有多于一个请求/应答事务 (应用程序将需要多个线程) ,那么它可以向服务器打开多个通道. 如图所示[图4](#implicitAckTimeline)应答消息用于确认请求消息,随后的请求确认前面的应答. 请注意,我们看到了一种非常类似的方法. *并发逻辑信道*-在稍后的部分中,作为提高停止和等待可靠性机制的性能的一种方式. 

<figure class="line">
	<a id="implicitAckTimeline"></a>
	<img src="figures/f05-16-9780123850591.png" width="200px"/>
	<figcaption>Timeline for a reliable RPC protocol using implicit
	acknowledgment.</figcaption>
</figure>

RPC必须解决的另一个复杂问题是,服务器可能需要任意长的时间来生成结果,更糟糕的是,在生成应答之前可能崩溃. 请记住,我们讨论的是服务器确认请求之后但在发送回复之前的时间. 为了帮助客户端区分慢服务器和死服务器,RPC的客户端可以定期发送"您还活着吗?"消息到服务器,服务器端用ACK进行响应. 或者,服务器可以向客户端发送"我仍然活着"的消息,而客户端没有首先请求它们. 该方法更具有可扩展性,因为它将更多的客户端负担 (管理超时定时器) 放到客户端上. 

RPC的可靠性可以包括已知的属性*至多语义*. 这意味着对于客户端发送的每个请求消息,至多一个消息的副本被传递到服务器. 每次客户端调用远程过程时,该过程最多在服务器机器上调用一次. 我们说"最多一次"而不是"完全一次",因为网络或服务器机器总是可能出现故障,使得甚至无法传递请求消息的一个副本. 

为了实现最多一次的语义,服务器端的RPC必须识别重复请求 (并忽略它们) ,即使它已经成功地响应了原始请求. 因此,它必须保持一些识别过去请求的状态信息. 一种方法是使用序列号来识别请求,因此服务器只需要记住最近的序列号. 不幸的是,这会将RPC限制为一次一个未完成的请求 (对给定服务器) ,因为在发送具有下一个序列号的请求之前,必须完成一个请求. 再次,渠道提供了一个解决方案. 服务器可以通过记住每个信道的当前序列号来识别重复请求,而不将客户端限制为一次一个请求. 

与大多数声音一样明显,并非所有RPC协议都支持这种行为. 一些支持一种被戏称为语义的语义. *零或更多*语义;即,在客户端上的每次调用导致远程过程被调用零次或多次. 不难理解,对于每次调用时都改变某些局部状态变量 (例如,递增计数器) 或具有某些外部可见副作用 (例如,发射导弹) 的远程过程,这将如何导致问题. 另一方面,如果正在调用的远程过程是*幂等元*-多个调用的效果与一次调用相同,那么RPC机制最多不需要支持一次语义;更简单 (可能更快) 的实现就足够了. 

与可靠性的情况一样,RPC协议可能实现消息分段和重新组装的两个原因是,它不是由底层协议栈提供的,或者它可以由RPC协议更有效地实现. 考虑RPC在UDP/IP之上实现的情况,并依赖于IP进行碎片化和重组. 如果甚至一条消息的一个片段未能在一定时间内到达,则IP丢弃确实到达的片段,并且消息有效地丢失. 最终,RPC协议 (假设它实现可靠性) 将超时并重传消息. 相反,考虑RPC协议,实现其自己的碎片和重新组装和积极的ACK或NACKs (否定承认) 单个片段. 丢失的片段将被更快地检测和重传,并且只有丢失的片段将被重传,而不是整个消息. 

### 同步与异步协议

描述协议的一种方法是它是否是*同步的*或*异步的*. 这些术语的确切含义取决于协议层次结构在哪里使用它们. 在传输层,将它们看作定义光谱的极端而不是两个相互排斥的备选方案是最准确的. 沿着频谱的任何点的关键属性是发送过程在发送消息操作返回之后知道多少. 换句话说,如果我们假设一个应用程序调用一个`send`在传输协议上的操作,那么应用程序对操作成功的确切了解是什么?`send`操作返回?

在*异步的*频谱结束时,应用程序什么时候都不知道. `send`返回. 它不仅不知道消息是否被它的对等方接收,而且它甚至不知道消息是否已经成功离开本地机器. 在*同步的*频谱结束`send`操作通常返回一条回复消息. 也就是说,应用程序不仅知道它发送的消息是由其对等方接收的,而且还知道该对等方已经返回了答案. 因此,同步协议实现请求/应答抽象,而如果发送方希望能够发送许多消息而不必等待响应,则使用异步协议. 使用这个定义,RPC协议显然是同步协议. 

虽然我们在这一章中没有讨论过,但这两个极端之间有一些有趣的观点. 例如,传输协议可能实现`send`因此,它阻塞 (不返回) 直到在远程机器上成功接收到消息,但在发送方在该机器上的对等点实际处理并响应消息之前返回. 这有时被称为*可靠数据报协议*.

## RPC实现 (SUNRPC,DCE) 

现在,我们将我们的讨论转化为RPC协议的一些示例实现. 这些将用来突出协议设计者做出的一些不同的设计决策. 我们的第一个例子是SUNRPC,一种广泛使用的RPC协议,也称为开放网络计算RPC (ONC RPC) . 我们的第二个例子,我们将称之为DCE-RPC,是分布式计算环境 (DCE) 的一部分. DCE是一组用于构建分布式系统的标准和软件,由开放软件基金会 (OSF) 定义,该基金会最初包括IBMㄡ数字设备公司和惠普 (Hewlett-Packard) ;今天,OSF取名为开放组 (Open Group) . 这两个例子代表了RPC解决方案空间中有趣的替代设计选择. 

<figure class="line">
	<a id="sunrpc"></a>
	<img src="figures/f05-17-9780123850591.png" width="100px"/>
	<figcaption>Protocol graph for SunRPC on top of UDP.</figcaption>
</figure>

### 日照

SunRPC成了*事实上的*由于它在Sun工作站中的广泛分布以及在Sun流行的网络文件系统 (NFS) 中扮演的中心角色,所以它是标准的. IETF随后将其作为ONC RPC名称下的标准互联网协议. 

可以通过几种不同的传输协议实现SUNRPC. [图5](#sunrpc)说明了在UDP上实现SunRPC时的协议图. 正如我们在本节前面提到的,严格的层论者可能会反对在传输协议上运行传输协议的想法,或者认为RPC必须是传输协议之外的东西,因为它看起来"高于"传输层. 实际上,在现有传输层上运行RPC的设计决策非常有意义,这一点在下面的讨论中将显而易见. 

SunrPC使用两层标识符来标识远程过程: 32位程序号和32位进程号.  (还有一个32位的版本号,但是在下面的讨论中我们忽略了这个. ) `x00100003`在这个程序中`getattr`IS过程`1`,`setattr`IS过程`2`,`read`IS过程`6`,`write`IS过程`8`等等. 在SUNRPC请求消息的报头中发送程序编号和过程编号,其字段显示在[图6](#sunrpc-format). 服务器可以支持多个程序号,负责调用指定程序的指定过程. SunRPC请求实际上表示在请求被发送到的特定机器上调用指定程序和过程的请求,即使相同的程序号可以在同一网络中的其他机器上实现. 因此,服务器的机器的地址 (例如,IP地址) 是RPC地址的隐式第三层. 

<figure class="line">
	<a id="sunrpc-format"></a>
	<img src="figures/f05-18-9780123850591.png" width="400px"/>
	<figcaption>SunRPC header formats: (a) request; (b) reply.</figcaption>
</figure>

不同的程序编号可能属于同一台机器上的不同服务器. 这些不同的服务器具有不同的传输层解复用密钥 (例如,UDP端口) ,其中大多数不是公知的数字,而是动态分配的. 这些DEMUX键被称为*传输选择器*. 想要与特定程序进行通信的SunRPC客户端如何确定使用哪个传输选择器来到达相应的服务器?解决方案是指派一个众所周知的地址. *只有一个*远程机器上的程序,让该程序处理告诉客户端使用哪个传输选择器来到达机器上的任何其他程序的任务. 这个SUNRPC程序的原始版本称为*端口映射器*,它只支持UDP和TCP作为底层协议. 它的程序号是`x00100000`其著名的港口是`111`. 从端口映射器演化而来的RPCBIN支持任意的底层传输协议. 当每个SunRPC服务器启动时,它调用一个RPCBIND注册过程,在服务器自己的主机上注册它的传输选择器和它所支持的程序号. 远程客户端然后可以调用RPCBACK查找过程查找特定程序号的传输选择器. 

为了使这更具体,请考虑使用端口映射器与UDP的示例. 向NFS发送请求消息`read`过程中,客户端首先在已知的UDP端口上向端口映射器发送请求消息. `111`问那个程序`3`调用程序编号`x00100003`到NFS程序当前驻留的UDP端口. 客户端然后发送带有程序编号的SUNRPC请求消息. `x00100003`程序编号`6`对于这个UDP端口,在该端口监听的SUNRPC模块调用NFS`read`程序. 客户机还缓存程序到端口号映射,以便它不必每次希望与NFS程序对话时都返回到Port Mapper. 

> 实际上,NFS是一个非常重要的程序,它已经被赋予了自己著名的UDP端口,但是为了便于说明,我们假装不是这样. 

为了将应答消息与对应的请求进行匹配,以便RPC的结果可以返回到正确的调用者,请求和应答消息头都包含`XID` (事务ID) 字段,如[图6](#sunrpc-format). 一`XID`是唯一的事务ID,仅由一个请求和相应的应答使用. 在服务器成功回复给定请求之后,它不记得`XID`. 正因为如此,SunRPC不能保证最多一次语义. 

SUNRPC的语义细节取决于底层传输协议. 它没有实现它自己的可靠性,所以只有底层传输是可靠的才是可靠的.  (当然,任何运行在SunRPC上的应用程序也可以选择实现它自己的高于SunRPC级别的可靠性机制. ) 发送大于网络MTU的请求和回复消息的能力也取决于底层传输. 换言之,当涉及到可靠性和消息大小时,SunRPC不尝试改进底层传输. 由于SunRPC可以运行在许多不同的传输协议上,因此在不使RPC协议本身的设计复杂化的情况下,它提供了相当大的灵活性. 

返回到SUNRPC报头格式[图6](#sunrpc-format)请求消息包含可变长度`Credentials`和`Verifier`字段,客户端使用这两个字段向服务器进行自身身份验证,即提供客户端有权调用服务器的证据. 客户端如何向服务器进行自身身份验证是一个普遍问题,任何希望提供合理安全级别的协议都必须解决这个问题. 本主题将在另一章中详细讨论. 

### DCE-RPC

DCE-RPC是DCE系统的核心RPC协议,是Microsoft的DCOM和ActiveX下RPC机制的基础. 它可以与另一章中描述的网络数据表示 (NDR) 存根编译器一起使用,但是它也充当通用对象请求代理体系结构 (CORBA) 的底层RPC协议,CORBA是构建分布式ㄡ面向对象系统的行业标准. 

DCE-RPC,像SunRPC一样,可以在包括UDP和TCP在内的几种传输协议之上实现. 它还与SunRPC类似,因为它定义了两级寻址方案: 传输协议解复用到正确的服务器,DCE-RPC分派到该服务器导出的特定过程,客户机咨询"端点映射服务" (类似于SunRPC的Port Mapper) 来leaRN如何到达一个特定的服务器. 然而,不像SUNRPC,DCE-RPC最多实现一次调用语义.  (实际上,DCE-RPC支持多个调用语义,包括与SunRPC类似的幂等语义,但最多一次是默认行为. ) 两种方法之间还有一些其他的不同,我们将在以下段落中强调这一点. 

<figure class="line">
	<a id="dce"></a>
	<img src="figures/f05-19-9780123850591.png" width="200px"/>
	<figcaption>Typical DCE-RPC message exchange.</figcaption>
</figure>

[图7](#dce)给出典型消息交换的时间线,其中每个消息由其DCE-RPC类型标记. 客户端发送一个`Request`消息,服务器最终以`Response`消息,客户端确认 (`Ack`) 响应. 然而,客户机不定期地发送请求消息,而不是服务器确认请求消息. `Ping`消息到服务器,该服务器以`Working`消息以指示远程过程仍在进行中. 如果服务器的回复很快收到,没有`Ping`发送. 虽然未在图中显示,但也支持其他消息类型. 例如,客户端可以发送`Quit`向服务器发出消息,要求它中止正在进行中的较早调用;服务器用`Quack` (退出确认) 消息. 此外,服务器可以响应`Request`带A的消息`Reject`消息 (指示调用已被拒绝) ,并且它可以响应`Ping`带A的消息`Nocall`消息 (指示服务器从未听到调用方) . 

DCE-RPC中的每个请求/应答事务发生在AN上下文中. *活动*. 活动是一对参与者之间的逻辑请求/应答通道. 在任何给定的时间,在给定的信道上只能有一个消息事务活动. 与上面描述的并发逻辑通道方法一样,如果应用程序想要同时在它们之间有一个以上的请求/应答事务,则它们必须打开多个通道. 消息所属的活动由消息标识. `ActivityId`字段. 一`SequenceNum`字段然后区分作为同一活动的一部分的调用;它与SunRPC的目的相同. `XID` (事务ID) 字段. 与SunRPC不同,DCE-RPC跟踪作为特定活动的一部分使用的最后一个序列号,以确保最多一次的语义. 为了区分服务器机重新启动之前和之后发送的答复,DCE-RPC使用`ServerBoot`字段保存机器的引导ID. 

与SunRPC不同的另一个设计选择是支持RPC协议中的碎片化和重新组装. 如上所述,即使诸如IP之类的底层协议提供分段/重组,作为RPC的一部分实现的更复杂的算法也能够导致更快的恢复以及当分段丢失时降低带宽消耗. 这个`FragmentNum`字段唯一标识构成给定请求或答复消息的每个片段. 每个DCE-RPC片段被分配一个唯一的片段数 (0, 1, 2,3,等等) . 客户端和服务器都实现了一种选择性确认机制,其工作原理如下.  (我们根据客户机向服务器发送分段请求消息来描述机制;当服务器向客户机发送分段响应时,相同的机制也适用. ) 

首先,组成请求消息的每个片段都包含唯一的`FragmentNum`以及指示该分组是否是呼叫的片段的标志 (`frag`或调用的最后片段 () ;适合于单个分组的请求消息带有标志. 服务器知道它已经收到完整的请求消息,当它有数据包,并且在片段号中没有空隙. 第二,响应每个到达的片段,服务器发送一个`Fack` (片段确认) 消息到客户端. 此确认标识服务器已成功接收的最高片段数. 换句话说,确认是累积的,就像在TCP中一样. 此外,服务器选择性地确认其接收到的任何更高的碎片编号. 它使用一个位向量来识别这些无序片段,相对于它接收到的最高有序片段. 最后,客户端通过重发缺失的片段来进行响应. 

[图8](#fack)说明了这一切是如何工作的. 假设服务器已经成功地接收到通过数字20ㄡ加上片段23, 25和26的片段. 服务器以一个`Fack`它将片段20识别为最高的有序片段,再加上一个位向量 (`SelAck`) $第三 (23=20+3) $,第五元 (25=20+5) $,第六元 (26=20+6) $$位开启. 为了支持 (几乎) 任意长的位向量,向量的大小 (在32位字中测量) 给出在`SelAckLen`字段. 

<figure class="line">
	<a id="fack"></a>
	<img src="figures/f05-20-9780123850591.png" width="500px"/>
	<figcaption>Fragmentation with selective acknowledgments.</figcaption>
</figure>

给定DCE-RPC对非常大消息的支持`FragmentNum`字段是16位长,这意味着它可以支持64K个片段ℴℴ协议不宜尽可能快地爆炸构成消息的所有片段,因为这样做可能会超出接收器. 相反,DCE-RPC实现了一种非常类似于TCP的流控制算法. `Fack`消息不仅确认接收到的片段,还通知发送者现在可以发送多少片段. 这就是目的`WindowSize`字段在[图8](#fack)与TCP的用途完全相同`AdvertisedWindow`字段除了计数片段而不是字节. DCE-RPC还实现了一种类似于TCP的拥塞控制机制,考虑到拥塞控制的复杂性,一些RPC协议通过避免碎片化来避免拥塞控制可能并不奇怪. 

总而言之,设计者在设计RPC协议时有相当多的选项可供他们选择. SunRPC采用更简约的方法,除了定位正确的过程和识别消息的基本要素之外,对底层传输的添加相对较少. DCE-RPC增加了更多的功能,在某些环境中有可能以较高的复杂性为代价提高性能. 
