---
layout: post
title: 基于RTSP协议下的NAT穿越分析与测试
feature-img: "img/sample_feature_img.png"
---
本文档主要分析在视频监控中进行P2P通信时穿越NAT所要遇到的主要问题和解决方案，文档中会具体分析穿越NAT时可能遇到的主要情况，并且根据这些情况设计解决方案和实验方法。为后面实现P2P的连接打好坚实的基础。
![Google语音通信的数据流动图](/img/NAT_travesal/all_frame_two.png)
<!--more-->

##Introduction

本文主要讨论的是NAT的穿越策略，大部分的算法主要依据于STUN的描述来进行处理。文章的开始会对P2P在监控系统穿越NAT的需求进行详细的分析，特别是分析在RTSP协议当中如何保证NAT穿越之后的正常运行。文章最后会根据前面的分析计划出一个详细的测试计划，根据这个测试计划进行技术测试。

本文主要分为四个部分。

* 第一部分首先会对NAT穿越技术的一个总体介绍，然后分析在我们项目中可能遇到的技术问题。
* 第二部分是对NAT穿越技术的详细探讨，同时根据STUN技术细节对NAT进行分类。
* 第三部分对穿越NAT的技术进行全方位的规划，详细列举测试方法和期望的结果。
* 第四部分是对这些测试进行深入的总结和分析，对前面的内容进行分析改进，提出最后的解决方案。

##NAT穿越技术和问题

穿越NAT的最终目标是为了在两台计算机之间建立直接连接，而不用通过服务器来转发数据，从而解放服务器。在理想状态下，穿越NAT之后，两台计算机将会分别得到对方可直接通信的IP地址和端口，接下来的事情就简单了。

在我们的项目当中，两台计算机之间主要进行的是RTSP协议的通信。在RTSP协议的开始建立过程中需要两台计算机先建立起TCP的通信，进行一些会话的协商，以便后面使用UDP或者TCP传送媒体数据。所以实际上在我们的项目当中，我们需要通过穿越NAT得到TCP和UDP两种可以连接的IP和端口，但是TCP的穿越非常的麻烦，可能最好只能使用UDP穿越。

###UDP的NAT穿越

对于UDP穿越NAT，现目前的应该当中有比较规范的穿越技术。这些穿越技术主要使用的是”反向连接技术”和 ”STUN”技术[1] 。这两种技术综合起来，一般情况下可以穿越互联网上面的大多数的NAT设备，取得良好的效果差不多能够达到92%的效果。但是这两种方法也不是万能的，总有一些NAT设备它不能够被这样简单的穿越，在文章的后面会详细讨论这个问题。

###TCP的NAT穿越

TCP穿越NAT有一篇比较好的论文《Characterization and Measurement of TCP Traversal through NATs and Firewalls》，里面介绍了大约五种TCP穿越NAT的方法，但是在实践和产品中很难应用。

对于TCP穿越NAT，在技术上面来说，这个问题非常的麻烦，由于TCP的连接特性，一般情况下是非常难以用TCP穿越NAT的。在现目前人们所做的研究当中，使用TCP穿越NAT的技术基本上分为两类。

第一类是NAT的规则非常的简单的情况进行穿越的算法。通过详细的会话测试集得到关于所属NAT设备的映射特性、过滤特性来判断这台计算要的端口是不是可以预测的。比如使用STUNT协议时它要求NAT映射的端口是连续的，或者同一端口发往同一IP地址的数据包被映射到相同的商品，只有在这些情况下才有可能被穿越[2] 。否则就没有办法，在实际过程当中，这种NAT设备比较少。

第二类主是使用复杂的概率和综合各种情况来进行端口猜测，实现复杂，失败率非常高，很不实用。

所以，综合来说，使用TCP穿越NAT的技术到现在并不成熟，也不像UDP一样有一个比较完善的解决方案。

##初步解决方案
理论上项目需要TCP和UDP同时穿越NAT,这样能够取得最好的效果，但是在实际过程当中，TCP的穿越基本上不可行，UDP的穿越也不是100%可以保证。所以在这个项目中会采取多各方法来保证通信。

对于UDP穿越，如果遇到实在不能够穿越的情况，那么只能使用一个中继服务器来进行数据的转发。对于TCP穿越，可在直接在实现RTSP的时候使用中继服务器来建立相关的连接会话，从而避免复杂的TCP穿越。最终的解决方案可以像Google Talk项目当中所使用的方法如下图所示：

![Google语音通信的数据流动图](/img/NAT_travesal/image001.jpg)

在实现当中，使用中继服务器的方法常常被称为TURN协议，而综合了STUN和TURN这两种方法的协议叫做ICE协议。ICE协议可以STUN和TURN协议的结果选择最好的连接方式，以达到最优的通信结果。
我个人认为，如果能够使用UDP来模拟TCP的连接，那么项目中的所有网络通信都可以是UDP数据，可以避免TCP的穿越。实现UDP模拟TCP并不复杂，这也许可以当一个备用的方案。

##客户端连接方法分析

在P2P建立过程当中，可能有时候有一些客户端和主机之间有时候隔着几层NAT，有时候隔着一层NAT，也有的时候它们之间在一个网络当中。还有的情况是一台客户端在NAT后面，一台客户端在公网上面。这些情况都需要具体考虑。在我们项目当中，一般情况下，两台连接的主机一台是显示客户端，一台是视频采集终端。

![Google语音通信的数据流动图](/img/NAT_travesal/table_p2p_connect_type.png)

###反向连接方法

当在显示客户端在公网当中，而视频采集终端在NAT之后的情况下，显示客户端不能够直接连接视频采集端，这个时候就可以使用反向连接方法。

具体来说，反向连接方法需要视频采集端一直连接到服务器，当公网当中的显示客户端需要请求视频信息的时候，直接向服务器发送连接请求。服务器再通知处于NAT之后的视频采集端主动连接显示客户端，这样就可以成功的建立一条P2P通道。

###基于RFC4787的NAT分类

RFC4787按照NAT把内部IP地址和端口映射的行为，将NAT分为三类。按照NAT过虑外部网络的数据包行为也将NAT分为三类。

####按端口映射行为分类

*	**Endpoint-Independent Mapping NAT**转换仅由私网源IP和源端口决定，即无论目的地址和目的端口是什么，一个私网源IP和源端口都固定映射到同一个公网源IP和源端口上。无论目的地址如何，相同的源IP有相同的映射结果。
*	**Address-Dependent Mapping NAT**转换由私网源IP、源端口和目的地址决定，即一个私网源IP、源端口和一个特定的目的地址将被映射到一个公网源IP和源端口。即使私网源IP和端口相同，但目的地址不同，会被映射到不同的公网地址和端口。
*	**Address and Port – Dependent Mapping NAT**转换由私网源IP、源端口、目的地址和目的端口决定，即由一个私网源IP、源端口去往相同目的和端口的报文被映射到一个特定的公网源IP和源端口。即使有相同的私网源IP、源端口和目的IP，目的端口不同也会有不同的映射表项。

####	按过虑行为分类

*	**Endpoint-Independent Filtering**只过虑不发往内部地址X:x(地址：端口)的流量，而不考虑其流量的源地址和源端口。
*	**Address – Dependent Filtering** 如果内部地址X:x没有先向外部地址Y发送流量的，NAT过虑外部地址Y发往内部地址X:x。即，只有X:x先发送流量到Y，Y才能发送流量到X:x。
*	**Address and Port – Dependent Filtering** 如果内部地址X:x没有先向外部地址Y:y发送流量的，NAT过虑外部地址Y:y发往内部地址X:x。换句话讲，只有X:x先发送流量到Y:y，Y:y才能发送流量到X:x。

##	基于STUN的NAT功能分类

NAT 主要分为两类一种是基本的NAT，另一种是NAPT (Network Address/Port Translator)。第一种NAT就是在地址转换过程当中 只转换IP地址，而不会改变端口，而第二种会同时改变端口，在现目前的应用情况下，NAPT的设备最为常见。为了与其它资料兼容，在这篇文章当中，所有的NAT基本上代表的是NAPT [5] 。

STUN执行过程当中，按NAT对端口的映射方式将其分为四种类型。

###	Full-cone NAT
这种NAT同时也称为one-to-one NAT。这种NAT的特点是一个内网的IP地址和端口被映射成一个外网的IP地址和端口(eAddr:ePort)。从内网中的iAddr:IPort端口所发送的任何数据都是都是通过这个外网固定eAddre: ePort发送出去的。同时任何外网的不同主机发送给这个NAT设备的eAddr:EPort的信息都将会转发到内网中iAddr: iPort当中去。[6] 

![Full-cone NAT](/img/NAT_travesal/image002.png)

###(Address)-Restricted-cone NAT

和Full-cone NAT一样，一个内网的IP地址和端口被映射成一个外网的IP地址和端口(eAddr:ePort)。从内网中的iAddr:IPort端口所发送的任何数据都是都是通过这个外网固定eAddre: ePort发送出去的。但是对于外网的主机，只有内网发送的目标主机发送的数据才能通过NAT的这个端口。这个目标主机的任意端口都可以通过这个端口。

![(Address)-Restricted-cone NAT](/img/NAT_travesal/image003.png)

###Port-restricted Cone NAT

和上面的两类非常相同，同样是端口一一映射。但是外网的主机只允许目标主机的目标端口的数据才能通过NAT到达内网的主机。

![Port-restricted Cone NAT](/img/NAT_travesal/image004.png)

###Symmetric NAT

这种NAT与前面三种有非常大的不同，内网中的主机所发送的数据只要是不同的外网主机NAT都为它分配一个不同的端口，而内网的主机即使使用的同一个端口，只要发送的目标是不同的主机。那么NAT都会为之重要分配一个端口。而对于外网的主机，NAT只接受固定的外网主机的固定端口访问。

![Symmetric NAT](/img/NAT_travesal/image005.png)

这一种类型的NAT在穿越的时候实际上几乎不可能的，不过幸好，由于这种NAT的实现比较复杂，所以在现实的应用当中比较少。

##STUN协议的工作原理

在网上的STUN协议都基本上出自2003年出版的RFC3489，全称为Simple Traversal of User Datagram Protocol(UDP) Through Network Address Translators(NATs)[4] 。 但是在2008年RFC5389对STUN协议进行了一些改动，这个时候的STUN的英文全称是Session Traversal Utilities for NAT。原本RFC3489声称STUN是一个完全的穿越NAT的解决方案，而在新的RFC5389中，称原本的RFC3489只是穿越NAT的其中一种工具。新的STUN最大的改变就是增加了对TCP穿越的支持[3] 。

对于STUN的TCP穿越，现在还没有具体研究，但是基于先前的分析，可预测其结果也应该不会有太大的改变。

STUN的主要工作原理就是利用外网的计算机来辅助本地的计算机穿越NAT，得到本地计算机的外网IP地址和端口。STUN在工作过程当中需要服务器有两个公网的IP地址，或者两台公网上面的服务器。其实经过对NAT的分类，我们就已经发现了，只有前面三种NAT才有可能穿越。Symmetric NAT不能够穿越。

为了更了的分析STUN的运行原理，图 3 5是一个STUN的解析图，在这个图当中，有两台客户计算机分别在NAT后面，在公网上面还有一个SERVER，它有两个IP地址。现在我们模拟STUN协议的运行过程。

* 第一步：Client A的3321端口向服务器的IP1发送UDP数据包。Client A的数据包被NAT转换之后，换成公网的IP地址，同时端口变成了1233。这个时候NAT的公网IP向服务器IP1发送数据。服务器的IP1把接受到的数据中IP地址和端口打包封装成一个数据包，然后发回给Client A（这个时候是NAT的1233端口，NAT再转发给Client A）。正常情况下这个过程一定会执行成功，如果出现问题，那么多半是这个网络有问题，或者SERVER 并不存在。

* 第二步：Client A收到了服务器IP1返回的自己在公网当中的IP地址和端口，将它与自己发送的IP地址和端口进行对比，如果发现这两个一样。那么就可以断定自己也在公网当中，这个时候看Client B的情况，如果Client B在内网当中，那么就可以使用反向连接方法。如果Client B也在公网当中，那么就可以直接连接。Client A发现自己的IP地址和端口和公网的不一样，那么就需要NAT穿越了。


* 第三步：Client A知道自己在NAT之后了，这个时候就需要与服务器相互配合来确定自己的NAT类型。Client A先向服务器IP1发送一个请求，请求服务器使用IP2和不同的端口向Client A的公网地址和相同端口发送数据。

如果Client A收到了服务器IP2的不同端口的数据，那么说明NAT是一个Full Cone NAT，这个时候就可以通知Client B进行连接，基本上NAT穿越成功了。不过遗憾的是在现实的应用当中Full Cone NAT实在非常少，所以一般情况下，很难接收到数据。

如果Client A没有收到服务器IP2的数据，那么说明NAT有可能是Restricted Cone NAT也可能是Port Restricted Cone NAT，还有可能是Symmetric NAT。这个时候也需要做继续探测。

* 第四步：Client A向服务器的IP2发送一个数据包，服务器IP2收到这个数据包之后，将Client A的公网IP和端口打包封装，并返回给Client A。在连接上面，这个过程一定能够成功。当Client A收到这个包之后，就与先前发往服务器IP1返回的包的端口进行对比。
* 
如果两个端口是不同的，那么可以确定这个NAT是Symmetric NAT，这个时候P2P直接不可能了，只能使用中继服务器来转发数据了。

如果两个端口是相同的，那么可以确定这个NAT，有可能是Restricted Cone NAT也可能是Port Restricted Cone NAT。

* 第五步：Client A向服务器IP1发送一个请求，让服务器IP1以不同的端口像Client A的公网IP和端口回复一个数据。
* 
![STUN解析图](/img/NAT_travesal/image006.png)

如果Client A收到了 回复数据包，那么可以说明这个NAT是Restricted Cone NAT。

如果没有收到，那么可以说明这个NAT是Port Restricted Cone NAT。

Client A和Client B一旦确定了自己的NAT类型和公网使用的IP地址和端口。那么Client A和Client B就可以直接向对方发送数据包，一旦发送了这样的数据包，那么它们就在各自的NAT上面打了一个洞，供对方的数据进来。

##实验设计与需求
为了保证实验的准确和有效率的进行，在这里我们详细讨论实验所需要的硬件设备、软件以及所需要做的实验。从总体上来看，我们的实验一共分为三类。

1.	测试STUN协议的连接方法，还包括当客户端在公网，视频采集端在NAT之后时所需要用的反向连接方法。
2.	测试当STUN协议不可达时，中继服务器的行为。
3.	测试多层NAT之下的NAT穿越。

###实验设计
综上所述表 4.1是所有需要做的STUN协议实验。根据各种NAT的类型行为推断，采用UDP打洞方式如果想要建立连接，那么只有两种情况下不可能[9] 。

* 两方都在Symmetric NAT后面。
* 一方在Symmetric NAT后面，一方在Port Restricted Cone NAT后面。

![实验差异](/img/NAT_travesal/table_diff.png)

对于STUN协议实验，又分为两类。

1.	测试NAT的类型，如图 4 1就是STUN在做NAT类型测试时的实验网络拓扑图。
2.	在测试了NAT的类型之后，测试两台计算机之间的直接连接，在这个时候测试反向连接方法，图 4 2所示的两台计算机的网络拓扑图，可以把Client B的NAT取消之后，就可以测试反向连接方法。、

![STUN的NAT类型测试](/img/NAT_travesal/image007.png)

![两台计算机之间的连接](/img/NAT_travesal/image008.png)

除此之外，经过对NAT的分类分析，可以预见，在经过多层的NAT的情况之下，这些NAT所表现出来的特性和规则最严格的NAT行为是一样的，下表描述的这样的预测，不过要经过一些简单的测试。。

![两台计算机之间的连接](/img/NAT_travesal/table_nat_inter.png)

但是不管怎么猜测，依然需要用实验来证明其正确性。下图是多层测试时所需要使用的网络拓扑图。

![两台计算机之间的连接](/img/NAT_travesal/image009.png)

###实验所需要的材料

由于实验条件有限，所有的实验全部以模拟为主。只要能够达到预期效果，就算实验成功。综合上面的实验计划，所需要的材料如下。

* STUN协议开源软件。
* 反向连接软件。
* 计算机三台：不需要服务器有两个IP，只需要测试时其它计算机能达到模拟效果就可以。
* 网线四根：在实验过程当中，最大的连接情况需要四根网线。
* 可以设置NAT规则的路由器两个。

##实验以及结果

**前期准备**

![两台计算机之间的连接](/img/NAT_travesal/table_complete.png)

###实验一：使用路由器模拟网络环境

**实验综合信息**

![两台计算机之间的连接](/img/NAT_travesal/table_route.png)

![两台计算机之间的连接](/img/NAT_travesal/image010.png)

**实验结果**

![两台计算机之间的连接](/img/NAT_travesal/image011.png)

以上内容是StunClient输入的信息，这些信息显示，在局域网内的NAT行为如下表所示。

![两台计算机之间的连接](/img/NAT_travesal/table_TP_LINK_TL.png)

###实验二，公网STUN服务器的测试

由于在实验阶段，创建公司的STUN服务器比较麻烦，同时没有具体的方案，所以选择了对国外的免费STUN服务器的测试[8] 。

**实验准备**

![两台计算机之间的连接](/img/NAT_travesal/table_stun_test.png)

**实验结果**

![两台计算机之间的连接](/img/NAT_travesal/table_stun_server.png)

![两台计算机之间的连接](/img/NAT_travesal/table_stun_result.png)


除此之外，我还在家里对这些服务器进行过测试，而NAT的行为与公司表现情况一样。但是我家的路由器和公司的路由器用的一样的，都是TP-Link类型的。实验一证明，在内网TP-Link的NAT所表现的行为和国外STUN服务器是一样的。

同时，除我们本身的路由器之外，从公司到国外一定还经过了许多NAT，但是这些NAT都没有改变这些行为，依据如此，我可以推测，现在的NAT基本上过虑的行为大都表现在自家的路由器上面。其它的路由器并没有改变其行为，或者说，其它的路由器的NAT规则，只比自家的路由器规则还弱。

事实上，随着P2P技术应用越来越普及，越来越多的系统或网络设备所带的NAT都支持NAT穿越，据实验统计，在2005年前使用的windows OS所带的NAT有94%以上都支持UDP穿越。思科、3COM等主流硬件广商设备中的NAT 100%支持UDP打洞方式，随着P2P的发展，这个数字肯定会增加 [9] 。

###实验三，基于STUN协议的客户端点对点互联实验

这一次的实验总体来说分为两部分，第一部分是局域网内测试，一种是公网测试。在实验之前，经过比较长时间的研究和对比，我最终选择了libjingle来进行测试，具体是使用是其中的peerconnection这个示例来进行的测试。

**实验准备**

![两台计算机之间的连接](/img/NAT_travesal/table_p2p.png)

![两台计算机之间的连接](/img/NAT_travesal/table_result.png)

在这一次的实验当中，不仅仅测试了点对点连接的理论，同时也测试了NAT的穿越成功率，在上面的测试过程当中，我尽量测试多个地方的网络，其中对于公网的测试，我测试了几点地区之间的网络连接，并没有遇到不能够连接的情况，总体来说效果比较好。上述测试的所有NAT类型都是和局域网内的类型一样。

不过有一点遗憾的是，在这一次的测试过程当中 ，客户端B穿越NAT总是不能穿越成功，可是它的NAT类型是一样的，而我也不知道具体原因。

##总结

本文，主要基于STUN对P2P穿越NAT做了比较详细的分析，包括NAT的分类，可能穿越的NAT类型，之后通过实验的方法验证了NAT穿越的过程中所可能的问题。为后面的真正项目开发做了非常充分的准备。

本文档也有不足之处，第一个是对于不同类型的NAT的穿越并没有做到，只是限于实验条件。不过在[10] 中，详细测试了18种NAT设备，在不同操作系统，不同应用当中 的P2P连接情况。最后的测试结果指出，大部分的NAT设备为Independent Mapping、Port Dependent Filter也就是说支持P2P穿越。

第二个不足之处就是有一台计算机并不是和一般的计算机在同一个网段，NAT类型和我们都一样，但是结果不管怎么样，这台计算机都不能和其它计算机建立P2P连接。我不知道具体的原因，初步认为是由于防火墙的问题，后面我仍然会研究这个问题。



##Refrence
* **[1]** 基于NAT穿越技术和P2P通信方案的研究与实现，徐向阳、韦昌法  2007
* **[2]** v关于TCP穿越NAT技术的研究与分析，马义涛、薛质、王轶骏 2008
* **[3]** RFC 5389 http://www.ietf.org/rfc/rfc5389.txt
* **[4]** 	RFC 3489 http://www.ietf.org/rfc/rfc3489.txt
* **[5]** 	P2P原理和常见的实现方式 http://www.cppblog.com/peakflys/archive/2013/01/25/197562.html
* **[6]** 	Network Address Translation https://en.wikipedia.org/wiki/Network_address_translation
* **[7]** 	STUNTMAN http://www.stunprotocol.org/
* **[8]** 	Free Stun Server List https://gist.github.com/zziuni/3741933
* **[9]** 	远程会诊系统中点对点多媒体通讯模块的设计与实现，包宝音青 2010
* **[10]** 	Test Report for P2P-Friendly Properties of Network Address Translators (NAT)，工业技术研究院，2007

