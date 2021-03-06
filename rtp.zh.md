
# {{Paj.Toe}}

在分组交换的早期,大多数应用都与数据的移动有关: 访问远程计算资源ㄡ传输文件ㄡ发送电子邮件等. 然而,至少早在1981年,就开始进行实验以承载实时流量,例如数字化语音样本. 超过分组网络. 我们要求一个应用程序"实时"时,它有强烈的要求,及时交货的信息. Internet电话,或IP语音 (VoIP) ,是一个实时应用程序的经典示例,因为如果需要超过一秒钟的时间来获得响应,则不能轻松地与某人进行对话. 正如我们将很快看到的,实时应用程序对传输协议提出了一些特定的要求,而这些要求在本章中讨论的协议还没有很好地满足. 

<figure class="line">
	<a id="vat"></a>
	<img src="figures/f05-21-9780123850591.png" width="400px"/>
	<figcaption>User interface of a vat audioconference.</figcaption>
</figure>

包括视频ㄡ音频和数据的多媒体应用有时分为两类: *互动式*应用和*流动*应用. 一个早期和一个流行的互动类的例子是`vat`多方音频会议工具,常用于支持IP多播的网络. 典型的控制面板`vat`会议显示在[图1](#vat). 网络电话也是一种交互式应用,可能是写作时使用最广泛的一种. 基于Internet的多媒体会议应用是另一个例子. 现代即时通讯应用程序也使用实时音频和视频. 这些是具有最严格的实时要求的应用程序. 

流式应用程序通常将音频或视频流从服务器传送到客户端,并且以RealAudio等商业产品为特征. 以YouTube为代表的流视频已经成为互联网上主流的交通方式之一. 由于流式应用程序缺乏人与人之间的交互,因此它们对底层协议提出了不太严格的实时要求. 然而,及时性仍然很重要ℴℴ例如,您希望视频在按下"播放"后立即开始播放,一旦它开始播放,迟来的包将导致它停止或造成某种视觉退化. 因此,虽然流式应用不是严格实时的,但它们仍然具有与交互式多媒体应用足够的共同点,从而保证考虑用于这两种类型的应用的共同协议. 

现在应该显而易见,实时和多媒体应用的传输协议的设计者在定义足够广泛的需求以满足非常不同的应用的需求方面面临真正的挑战. 它们还必须注意不同应用程序之间的交互,例如音频和视频流的同步. 我们将在下面看到这些关注如何影响在今天使用的RTP的实时实时传输协议的设计. 

许多RTP实际上源自最初嵌入在应用程序本身中的协议功能. 当`vat`应用程序首先被开发,它运行在UDP上,设计人员计算出需要哪些特性来处理语音通信的实时性. 后来,他们意识到这些特性对于许多其他的应用程序是有用的,并且定义了具有这些特性的协议,这变成了RTP. RTP可以运行在许多低层协议上,但仍然通常在UDP上运行. 这导致协议栈显示在[图2](#vat-stack). 请注意,我们因此在传输协议上运行传输协议. 没有反对这个的规则,事实上这是很有意义的,因为UDP提供了这样最低级别的功能,并且基于端口号的基本解复用恰好是RTP需要的作为起点. 因此,RTP不是在RTP中重新创建端口号,而是将解复用功能外包给UDP. 

<figure class="line">
	<a id="vat-stack"></a>
	<img src="figures/f05-22-9780123850591.png" width="300px"/>
	<figcaption>Protocol stack for multimedia applications using RTP.</figcaption>
</figure>

## 要求

通用多媒体协议最基本的要求是允许类似的应用程序彼此互操作. 例如,应该有两个独立实现的音频会议应用程序相互交谈. 这立即表明,应用程序最好使用相同的语音编码和压缩方法;否则,一方发送的数据对接收方将是不可理解的. 由于有很多不同的语音编码方案,每个方案在质量ㄡ带宽要求和计算成本之间都有自己的折衷,所以规定只能使用一个这样的方案可能是个坏主意. 相反,我们的协议应该提供一种方式,发送者可以告诉接收者它想要使用哪种编码方案,并且可能进行协商,直到识别出双方都可用的方案为止. 

正如音频一样,有许多不同的视频编码方案. 因此,我们看到,RTP可以提供的第一个共同功能是通信选择编码方案的能力. 注意,这也用于识别应用程序的类型 (例如,音频或视频) ;一旦我们知道正在使用什么编码算法,我们也知道正在编码什么类型的数据. 

另一个重要要求是使数据流的接收者能够确定接收的数据之间的定时关系. 实时应用程序需要将接收到的数据放入*回放缓冲区*平滑在网络传输期间可能引入到数据流中的抖动. 因此,数据的某种时间戳将是必要的,以使接收器能够在适当的时间回放. 

与单个媒体流的定时有关的是会议中多个媒体的同步问题. 这样的一个明显的例子是同步来自同一发送者的音频和视频流. 正如我们将在下面看到的,这是一个比单个流的回放时间确定稍微复杂的问题. 

要提供的另一个重要功能是分组丢失的指示. 注意,具有紧密延迟边界的应用程序通常不能使用像TCP那样的可靠传输,因为重传数据以纠正丢失可能导致数据包到达得太晚而无用. 因此,应用程序必须能够处理丢失的数据包,并且处理它们的第一步是注意到它们实际上丢失了. 例如,使用MPEG编码的视频应用可以在分组丢失时采取不同的动作,这取决于分组是来自I帧ㄡB帧还是P帧. 

分组丢失也是拥塞的潜在指标. 由于多媒体应用通常不在TCP上运行,因此他们也错过了TCP的拥塞避免特性. 然而,许多多媒体应用能够响应拥塞,例如,通过改变编码算法的参数来减少所消耗的带宽. 显然,为了实现这一功能,接收机需要通知发送机正在发生丢失,以便发送机可以调整其编码参数. 

跨多媒体应用程序的另一个常见功能是帧边界指示的概念. 在此上下文中的框架是特定于应用程序的. 例如,通知视频应用程序某一组分组对应于单个帧可能是有用的. 在音频应用程序中,标记一个"talkspurt"的开始是有帮助的,它是一个声音或单词的集合,之后是静默. 然后,接收器可以识别通话喷头之间的沉默,并使用它们作为移动播放点的机会. 这遵循这样的观察,即词之间的空格的稍微缩短或延长对于用户来说是不可感知的,而词本身的缩短或延长既是可感知的,也是令人讨厌的. 

我们可能希望加入协议的最后一个功能是识别比IP地址更方便用户的发送者. 如图所示[图1](#vat)音频和视频会议应用程序可以在它们的控制面板上显示字符串,因此应用程序协议应该支持这样的字符串与数据流的关联. 

除了协议所需的功能之外,我们还注意到一个附加要求: 它应该合理有效地利用带宽. 换句话说,我们不想引入许多额外的比特,这些比特需要以长报头的形式与每个分组一起发送. 其原因是,作为最常见的多媒体数据类型之一的音频分组往往很小,从而减少了用样本填充它们所需的时间. 长的音频分组意味着由于分组化而导致的高延迟,这对于感知的对话质量具有负面影响.  (这是选择ATM信元长度的因素之一. ) 由于数据分组本身是短的,所以大的报头意味着报头将使用相对大量的链路带宽,从而减少"有用"数据的可用容量. 我们将看到RTP的设计的几个方面都受到保持头短的必要性的影响. 

你可以争论一下刚才描述的每一个特性. *真正地*需要一个实时传输协议,你可以找到更多可以添加的. 这里的关键思想是通过为应用程序开发人员提供一组有用的抽象和构建块来简化他们的工作. 例如,通过将时间戳机制放入RTP,我们将实时应用程序的每个开发人员从发明自己的应用程序中解救出来. 我们还增加了两个不同的实时应用程序可以互操作的可能性. 

## RTP设计

现在我们已经看到了针对多媒体的传输协议的相当长的需求列表,下面我们来看一下为满足这些需求而指定的协议的细节. 这个协议,RTP,是在IETF中开发并广泛使用的. RTP标准实际上定义了一对协议,RTP和实时传输控制协议 (RTCP) . 前者用于多媒体数据的交换,而后者用于周期性地发送与特定数据流相关联的控制信息. 当在UDP上运行时,RTP数据流和相关的RTCP控制流使用连续的传输层端口. RTP数据使用偶数端口号,RTCP控制信息使用下一个更高的 (奇数) 端口号. 

因为RTP被设计成支持各种各样的应用,它提供了一种灵活的机制,通过这种机制可以开发新的应用,而不必重复修改RTP协议本身. 对于每一类应用程序 (例如,音频) ,RTP定义了一个*简介*一个或多个*格式*. 概要文件提供了一系列信息,确保对应用程序类的RTP报头中的字段有一个共同的理解,这在详细检查报头时是显而易见的. 格式规范解释了如何解释跟随RTP报头的数据. 例如,RTP报头可能只跟随一个字节序列,每个字节代表一个在前一个定义间隔之后的单个音频样本. 或者,数据的格式可能要复杂得多,例如,MPEG编码的视频流需要有大量的结构来表示所有不同类型的信息. 

RTP的设计体现了一种被称为*应用层成帧技术* (阿尔夫) 这一原理是克拉克和TeNeNoWoice在1990提出的一种设计新兴多媒体应用协议的新方法. 他们认识到,这些新的应用程序不太可能被现有的协议 (如TCP) 所服务,而且它们可能不会被任何一种"一刀切"的协议所服务. 这个原则的核心是相信应用程序能最大程度地理解自己的需求. 例如,MPEG视频应用程序知道如何最好地从丢失的帧中恢复,以及如果丢失了I帧或B帧,如何作出不同的反应. 同一应用程序还最了解如何分段数据以进行传输,例如,最好在不同的数据报中发送来自不同帧的数据,这样丢失的分组只损坏单个帧,而不是损坏两个. 正是由于这个原因,RTP将许多协议细节留给特定于应用程序的概要文件和格式化文档. 

### 标题格式

[图3](#rtp-hdr)显示RTP使用的报头格式. 前12个字节总是存在的,而贡献源标识符只在某些情况下使用. 在该标头之后,可以有可选的头扩展,如下所述. 最后,报头后面是RTP有效载荷,其格式由应用程序决定. 该报头的意图是只包含可能被许多不同应用程序使用的字段,因为对于单个应用程序非常特定的任何内容将仅在该应用程序的RTP有效负载中更有效地承载. 

<figure class="line">
	<a id="rtp-hdr"></a>
	<img src="figures/f05-23-9780123850591.png" width="500px"/>
	<figcaption>RTP header format.</figcaption>
</figure>

前两位是版本标识符,它包含在写入时部署的RTP版本中的值2. 您可能认为协议的设计者相当大胆地认为2位将足以包含RTP的所有未来版本,但是请记住,位在RTP报头中占优势. 此外,针对不同的应用程序使用概要文件使得需要对基本RTP协议进行许多修订的可能性较小. 在任何情况下,如果发现在版本2之外需要另一个版本的RTP,则可以考虑对报头格式进行更改,以便将来可能出现多个版本. 例如,在版本字段中的值为3的新的RTP头可以在标题中的其他地方有一个"Sudio"字段. 

下一个比特是*衬垫* (`P`"比特",它是由于某种原因填充了RTP有效载荷的情况下设置的. 例如,可以根据加密算法的要求填充RTP数据以填充一定大小的块. 在这种情况下,RTP报头ㄡ数据和填充的完整长度将由较低层协议报头 (例如,UDP报头) 传送,并且填充的最后一个字节将包含应该忽略多少字节的计数. 这说明了[图4](#rtp-pad). 注意,这种填充方法消除了对RTP报头中长度字段的任何需要 (从而达到保持报头短的目的) ;在没有填充的常见情况下,长度是从较低层协议中推导出来的. 

<figure class="line">
	<a id="rtp-pad"></a>
	<img src="figures/f05-24-9780123850591.png" width="600px"/>
	<figcaption>Padding of an RTP packet.</figcaption>
</figure>

这个*延伸* (`X`bit用于指示扩展报头的存在,扩展报头将为特定应用程序定义并遵循主报头. 这种报头很少使用,因为通常可以将特定于有效负载的报头定义为特定应用程序的有效负载格式定义的一部分. 

这个`X`位后面是一个4位字段,用于计算*贡献来源*,如果有任何包含在页眉中. 下面将讨论贡献源. 

我们在上面提到了对某种帧指示的频繁需要;这是由标记位提供的,标记位具有特定于配置文件的用途. 对于语音应用程序,它可以设置在一个TalkSpt的开头,例如. 7位有效载荷类型字段如下;它指示在该分组中承载什么类型的多媒体数据. 该字段的一个可能的用途是允许应用程序基于关于网络中的资源可用性的信息或关于应用程序质量的反馈从一个编码方案切换到另一个编码方案. 有效载荷类型的精确使用也由应用程序配置文件决定. 

注意,有效载荷类型通常不被用作将数据定向到不同应用程序 (或定向到单个应用程序中的不同流,例如视频会议的音频和视频流) 的解复用键. 这是因为这样的解复用通常在较低层 (例如,通过UDP,如在前一节中描述) 提供. 因此,使用RTP的两个媒体流通常使用不同的UDP端口号. 

序列号用于使RTP流的接收器能够检测丢失和错误排序的分组. 发送方仅为每个发送的分组增加值一个. 注意,与TCP相比,RTP在检测到丢失的分组时不做任何事情,TCP既校正了丢失 (通过重传) ,又将丢失解释为拥塞指示 (这可能导致它减小其窗口大小) . 相反,当分组丢失时,由应用程序决定该做什么,因为这个决定可能高度依赖于应用程序. 例如,视频应用程序可能决定当包丢失时最好的做法是重放正确接收的最后一帧. 一些应用程序也可能决定修改它们的编码算法,以减少带宽需求以响应损失,但这不是RTP的功能. 对于RTP来说,决定发送速率应该降低是不明智的,因为这可能使应用程序无用. 

时间戳字段的功能是使接收器能够以适当的间隔回放样本,并使不同的媒体流能够同步. 因为不同的应用程序可能需要不同的时间粒度,RTP本身不指定测量时间的单位. 相反,时间戳只是一个"滴答"的计数器,其中蜱之间的时间取决于使用中的编码. 例如,一个音频应用程序,每125万美元$MU $ S采样一次数据,可以使用该值作为其时钟分辨率. 时钟粒度是在应用程序的RTP配置文件或有效载荷格式中指定的细节之一. 

数据包中的时间戳值是表示时间的数值. *第一*生成数据包中的样本. 时间戳不是一天中的时间的反映;只有时间戳之间的差异是相关的. 例如,如果采样间隔是125$mu$$s,并且分组n+1中的第一个采样是在分组n中的第一个采样之后10ms生成的,那么这两个采样之间的采样时刻的数量是

{%Center %}时间包/TimeStimult{%EntCenter %}

$ $ = (10倍10 ^ {-3 }) / (125倍10 ^ {-6 }) =80 $ $

假设时钟粒度与采样间隔相同,则分组n+1中的时间戳将比分组n中的时间戳大80. 注意,由于诸如静默检测的压缩技术,可能已经发送了少于80个样本,但是时间戳允许接收器以正确的时间关系回放样本. 

同步源 (SSRC) 是唯一标识RTP流的单个源的32位数字. 在给定的多媒体会议中,每个发送方选择一个随机SSRC,并且期望在两个源选择相同值的不太可能的事件中解决冲突. 通过使源标识符与源的网络或传输地址不同,RTP确保了与底层协议的独立性. 它还使得具有多个源的单个节点 (例如,多个摄像机) 能够区分这些源. 当单个节点生成不同的媒体流 (例如,音频和视频) 时,不需要在每个流中使用相同的SSRC,因为在RTCP (下面描述) 中存在允许中间同步的机制. 

只有当RTP流通过混合器时才使用贡献源 (CSRC) . 混频器通过从多个源接收数据并作为单个流发送来减少会议的带宽要求. 例如,来自多个并发扬声器的音频流可以作为单个音频流进行解码和重新编码. 在这种情况下,混频器将自身列出为同步源,但也列出了贡献源,即对相关分组作出贡献的扬声器的SRC值. 

## 控制协议

RTCP提供与多媒体应用程序的数据流相关联的控制流. 该控制流提供三个主要功能: 

1.  对应用程序和网络性能的反馈

2.  一种关联和同步来自同一发送者的不同媒体流的方法. 

3.  一种在用户界面上显示发送者身份的方法 (例如,`vat`界面显示[图1](#vat)) 

第一功能对于检测和响应拥塞是有用的. 一些应用程序能够以不同的速率操作,并且可以使用性能数据来决定使用更积极的压缩方案来减少拥塞,例如,或者在拥塞很少时发送更高质量的流. 性能反馈也可以用于诊断网络问题. 

您可能认为第二个函数已经由RTP的同步源ID (SSRC) 提供了,但实际上不是. 正如已经注意到的,来自单个节点的多个相机可能具有不同的SSRC值. 此外,不要求来自同一节点的音频和视频流使用相同的SSRC. 因为SSRC值的碰撞可能发生,所以有必要改变流的SSRC值. 为了解决这个问题,RTCP使用了*规范名称*(CNAME)指派给发送方的,然后与发送方使用RTCP机制可能使用的各种SSRC值相关联. 

简单地关联两个流仅仅是中间同步问题的一部分. 因为不同的流可能具有完全不同的时钟 (具有不同的粒度,甚至具有不同数量的不准确度或漂移) ,所以需要有一种方法来精确地使流彼此同步. RTCP通过传送将每天的实际时间与在RTP数据包中携带的时钟速率相关的时间戳相关联的定时信息来解决这个问题. 

RTCP定义了许多不同的数据包类型,包括

-   发送者报告,它使活动发送者能够会话来报告发送和接收统计信息. 

-   接收者报告,接收者不使用发送者来报告接收统计信息. 

-   源代码描述,它承载cNeND和其他发送者描述信息. 

-   应用程序特定的控制包

这些不同的RTCP分组类型是通过下层协议发送的,正如我们已经注意到的,它们通常是UDP. 可以将几个RTCP分组打包到较低层协议的单个PDU中. 要求在每个低级PDU中至少发送两个RTCP分组: 一个是报告分组,另一个是源描述分组. 其他分组可以包括到由下层协议施加的大小限制. 

在进一步研究RTCP分组的内容之前,我们注意到,组播组的每个成员发送周期性控制通信量存在潜在问题. 除非我们采取一些措施来限制它,否则控制流量有可能成为带宽的重要消费者. 例如,在音频会议中,在任何时刻,最多不超过两个或三个发送方可能发送音频数据,因为没有必要每个人同时讲话. 但是,对于每个发送控制流量的人来说,并没有这样的社会限制,这可能是一个严重的问题,因为会议有数千人参加. 为了解决这个问题,RTCP具有一组机制,通过该机制,参与者随着参与者数量的增加而缩减报告频率. 这些规则有些复杂,但基本目标是: 将RTCP流量的总量限制为RTP数据流量的小百分比 (通常为5%) . 为了实现这个目标,参与者应该知道可能使用多少数据带宽 (例如,发送三个音频流的数量) 和参与者的数量. 他们从RTP以外的地方学习前者. *会话管理*,在本节结束时讨论,他们从其他参与者的RTCP报告中学习后者. 因为RTCP报告可能以非常低的速率发送,所以可能只能获得当前接收者数量的大致计数,但这通常就足够了. 此外,建议为活动发送方分配更多的RTCP带宽,前提是大多数参与者希望看到来自他们的报告,例如,找出谁在讲话. 

一旦参与者确定了RTCP流量可以消耗多少带宽,它就开始以适当的速率发送定期报告. 发送方报告和接收方报告的区别仅在于前者包含关于发送者的一些额外信息. 这两种类型的报告都包含关于最近报告期中从所有源接收到的数据的信息. 

发送方报告中的额外信息包括

-   包含生成报表时的实际时间戳的时间戳

-   RTP时间戳对应于生成报表时的时间

-   发件人自发送以来发送的数据包和字节的累积计数

注意,前两个量可用于实现来自相同源的不同媒体流的同步,即使这些流在其RTP数据流中使用不同的时钟粒度,因为它提供了将每天的时间转换为RTP时间戳的键. 

发送者和接收者报告包含从上一次报告以来已经听到的每个源的一个数据块. 每个块包含以下问题的统计数据: 

-   其SSRC

-   自从上次报告被发送以来从这个源丢失的数据分组的百分比 (通过比较接收的分组数量和预期的分组数量来计算;这个最后的值可以从RTP序列号确定) 

-   自第一次从该消息源丢失的数据包总数

-   从这个源接收到的最高序列号 (扩展到32位来解释序列号的包装) 

-   源的估计到达间抖动(通过比较接收分组的到达间间隔和传输时间的期望间隔来计算)

-   此源通过RTCP接收的最后实际时间戳

-   Delay自上次发件人报告通过RTCP接收此源

正如您所想象的,这些信息的接收者可以学习关于会话状态的各种事情. 特别地,他们可以看到其他接收方是否从某些发送方获得了比他们更好的质量,这可能是需要进行资源预约的指示,或者网络中存在需要注意的问题. 此外,如果发送方注意到许多接收机正在经历其分组的高损耗,那么它可能决定它应该降低其发送速率或者使用对损耗更具弹性的编码方案. 

我们将考虑的RTCP的最后一个方面是源描述包. 这样的分组至少包含发送者的SSRC和发送者的CNED. 规范名称以这样的方式导出,即生成可能需要同步的媒体流的所有应用程序 (例如,来自同一用户的单独生成的音频和视频流) 将选择相同的CNAME,即使它们可能选择不同的SSRC值. 这使得接收机能够识别来自同一发送者的媒体流. CNED最常见的格式是`host`是发送机器的完全限定域名. 因此,由用户名为"用户"的用户启动的应用程序是`jdoe`在机器上运行将使用字符串作为其名称. 在这个表示中使用的大的和可变的字节数将使它成为SSRC格式的不良选择,因为SSRC与每个数据包一起发送,并且必须实时处理. 允许在周期性RTCP消息中绑定到SSRC值的CSID能够为SSRC提供紧凑和高效的格式. 

在源描述包中可以包括其他项目,例如用户的实名和电子邮件地址. 它们用于用户界面显示和联系参与者,但对于RTP的操作来说不如CNAME重要. 

与TCP一样,RTP和RTCP是一对相当复杂的协议. 这种复杂性很大程度上是为了使应用设计者的生活更容易. 由于存在无限数量的可能应用,设计传输协议的挑战在于使其足够通用,以满足许多不同应用的广泛变化的需求,而不使协议本身无法实现. RTP在这方面已经证明是非常成功的,它构成了当今因特网上大多数实时多媒体通信的基础. 
