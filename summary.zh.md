
# {{ 页面标题 }}

本章描述了四种截然不同的端到端协议. 我们考虑的第一个协议是一个简单的解复用器,以UDP为代表. 所有这样的协议都是根据端口号向适当的应用程序进程发送消息. 它不会以任何方式增强底层网络的尽力而为服务模型 - 它只是为应用程序提供不可靠的无连接数据报服务. 

第二种类型是可靠的字节流协议,我们看到的这种类型的具体示例是TCP. 这种协议面临的挑战是从可能被网络丢失的消息中恢复,以与发送消息相同的顺序发送消息,并允许接收方对发送方进行流量控制. TCP使用基本的滑动窗口算法,通过广告窗口进行增强,以实现此功能. 该协议的另一个注意事项是准确的超时/重传机制的重要性. 有趣的是,即使TCP是单一协议,我们看到它采用了至少五种不同的算法 - 滑动窗口,Nagle,三次握手,Karn / Partridge和Jacobson / Karels--所有这些都可以对任何目的都有价值

到端协议. 我们研究的第三种传输协议是构成RPC基础的请求/回复协议. 此类协议必须将请求分派给正确的远程过程,并匹配对相应请求的回复. 

它们还可以提供可靠性,例如最多一次的语义,或者通过消息分段和重组来支持大的消息大小. 最后,我们研究了涉及多媒体数据 (如音频和视频) 并且需要实时传送的应用类传输协议. 这种传输协议需要提供帮助以恢复单个媒体流的定时信息并同步多个媒体流. 它还需要向上层 (例如,应用层) 提供关于丢失数据的信息 (因为通常没有足够的时间来重传丢失的分组) ,以便可以采用适当的应用特定恢复和拥塞避免方法. *为满足这些需求而开发的协议是RTP,其中包括一个名为的伴随控制协议*RTCP

## . 

进一步阅读毫无疑问,TCP是一个复杂的协议,实际上它具有本章未阐述的微妙之处;因此,本章的推荐阅读列表包括原始TCP规范. 我们包含此规范的动机并不是要填写缺少的细节,以便向您展示诚实到良好的协议规范. Birrell和Nelson的下一篇论文是关于RPC的开创性论文. *第三,Clark和Tennenhouse关于协议架构的论文引入了概念*应用层框架这激发了RTP的设计;

-   本文提供了对应用程序需求变化时设计协议所面临挑战的深刻见解. USC-ISI. *传输控制协议. *征求意见

-   793年,1981年9月. Birrell,A. 和B. Nelson. 实现远程过程调用. *ACM计算机系统交易*1984年2月2 (1) : 39-59. 

-   Clark,D. 和D. Tennenhouse. 新一代协议的架构考虑因素. *SIGCOMM '90研讨会的会议记录*,1990年9月,第200-208页. 
