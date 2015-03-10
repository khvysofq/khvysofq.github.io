---
layout: post
title: P2P穿透理论：ICE STUN TURN
feature-img: "img/sample_feature_img_2.png"
---
本文的主要工作是介绍ICE、STUN、TURN这几种协议。文章的目标并不是将这三个协议的内容翻译成中文，只是根据作者的理解，以及在Libjingle当中可能需要的重要部分而进行有选择性的分析。当然，如果有时间更加推荐去看原本的RFC文档，这一章的目标是分析，希望能够让对这些协议不了解的读者能够有一个基础的知识，能够为后面的Libjingle深入分析打好一个基础。本章分为四部分，第一部分介绍STUN协议，第二部分介绍TURN协议，第三部分介绍ICE协议，第四部分综合Libjingle和协议的内容进行一个简单的总结。每一部分都会有相应的实验，都会提取Libjingle中的相关代码来进行深入的分析。希望我的工作能够给你带来帮助和进步。
<!--more-->

Libjingle到底是什么呢？在最开始的项目当中，Libjingle是用于支持XMPP协议及其相关扩展的一个Google Talk当中的程序。后面Libjingle初移植到了WebRTC当中，成为了WebRTC的网络核心部分。事实上，在Google Talk当中，它是Google Talk的网络核心部分，在WebRTC当中也是一样的。那么WebRTC的网络核心是什么，是线程、套节字、消息、还有非常重要的P2P连接。

而P2P连接主要实现的就是ICE（RFC 5245）协议，ICE协议是一个综合的P2P连接解决方案，它包含了专门用于打洞的STUN（RFC5389）协议和当打洞不成功时使用的TRUN(RFC5766)转发协议。除了这两种方案之后，还定义了一系列的规则来保证P2P的连接，这就是ICE。而Libjingle当中的P2P就是主要实现了ICE协议的内容，更广泛的说，Libjingle的主要核心内容就是实现ICE。Libjingle在Google上没有太多的原理性描述细节，实际上ICE就是Libjingle的最好的文档。


##STUN 协议 RFC5389

STUN协议有两个版本一个是2003年3月出版的RFC3489（STUN - Simple Traversal of User Datagram Protocol (UDP) Through Network Address Translators (NATs),一个是在2008年10月出版的RFC5389（Session Traversal Utilities for NAT (STUN)）。而Libjingle支持的协议的RFC5389。在第一版的RFC当中，STUN被定义为一个使用UDP简单穿越NAT的解决方案，是一个完成的解决方案，而在新的RFC5389中STUN被定义为作为其它需要穿越NAT的协议的一个工具，比如ICE协议。

STUN协议的主要功能是用来打洞的，所谓打洞就是当两台计算分别处于不同的内网当中，他们之间想要直接连接，但是只知道自己在局域网内的IP地址，并不知道自己的外网IP地址，这样就根本不能够进行直接连接如图下图所示，所以这个时候就需要通过一定的方法来得到自己的外网IP地址，得到之后再通过一定的方法将两台计算机连接起来，这个过程就叫打洞，建立两台计算机之间的直接连接。

![generator](/img/ice/generator.png)

在这里，**知道对方的外网IP和端口之后，通过一定的方法将两台计算机连接**并没有我们相像中的那么简单。由于各种各样的原因，现在的NAT从打洞上面，可以分为四类NAT，而可能穿越的NAT只有其中三类。有一些NAT很容易打洞，而有一些NAT却很难打洞成功，具体的在下面的章节当中我们会详细讨论。STUN协议主要有两种功能：

1. 让一台计算机知道自己当前的数据包在外网（NAT之外）的IP地址和端口信息。
2. 当两台计算机打洞成功之后，可以用来检查和保持端口之间的连接状态。

### UDP的NAT穿越

对于UDP穿越NAT，现目前的应该当中有比较规范的穿越技术。这些穿越技术主要使用的是”反向连接技术”和
”STUN”技术@NAT_TRAVERSAL_1。这两种技术综合起来，一般情况下可以穿越互联网上面的大多数的NAT设备，取得良好的效果差不多能够达到92%的效果(来自Google的数据)。但是这两种方法也不是万能的，总有一些NAT设备它不能够被这样简单的穿越，在文章的后面会详细讨论这个问题。

### TCP的NAT穿越

对于TCP穿越NAT，在技术上面来说，这个问题非常的麻烦，由于TCP的连接特性，一般情况下是非常难以用TCP穿越NAT的。在现目前人们所做的研究当中，使用TCP穿越NAT的技术基本上分为两类。
第一类是NAT的规则非常的简单的情况进行穿越的算法。通过详细的会话测试集得到关于所属NAT设备的映射特性、过滤特性来判断这台计算要的端口是不是可以预测的。比如使用STUNT协议时它要求NAT映射的端口是连续的，或者同一端口发往同一IP地址的数据包被映射到相同的商品，只有在这些情况下才有可能被穿越@NAT_TRAVERSAL_2。否则就没有办法，在实际过程当中，这种NAT设备比较少。

第二类主是使用复杂的概率和综合各种情况来进行端口猜测，实现复杂，失败率非常高，很不实用。

所以，综合来说，使用TCP穿越NAT的技术到现在并没有一个比较好的解决方案。

### 客户端连接方法分析

在P2P建立过程当中，可能有时候有一些客户端和主机之间有时候隔着几层NAT，有时候隔着一层NAT，也有的时候它们之间在一个网络当中。还有的情况是一台客户端在NAT后面，一台客户端在公网上面。这些情况都需要具体考虑。在我们项目当中，一般情况下，两台连接的主机一台是显示客户端，一台是视频采集终端。

![table_nat](/img/ice/table_nat.png)

如上表所示，不同情况下的P2P连接情况，我们重点讨论的是两台计算机都在NAT之后的连接方案。

##基于RFC4787的NAT分类

RFC4787按照NAT把内部IP地址和端口映射的行为，将NAT分为三类。按照NAT过虑外部网络的数据包行为也将NAT分为三类。

### 按端口映射行为分类

- **Endpoint-Independent Mapping** NAT转换仅由私网源IP和源端口决定，即无论目的地址和目的端口是什么，一个私网源IP和源端口都固定映射到同一个公网源IP和源端口上。无论目的地址如何，相同的源IP有相同的映射结果。

- **Address-Dependent Mapping** NAT转换由私网源IP、源端口和目的地址决定，即一个私网源IP、源端口和一个特定的目的地址将被映射到一个公网源IP和源端口。即使私网源IP和端口相同，但目的地址不同，会被映射到不同的公网地址和端口。

- **Address and Port Dependent Mapping** NAT转换由私网源IP、源端口、目的地址和目的端口决定，即由一个私网源IP、源端口去往相同目的和端口的报文被映射到一个特定的公网源IP和源端口。即使有相同的私网源IP、源端口和目的IP，目的端口不同也会有不同的映射表项。

### 按过滤行为分类

- **Endpoint-Independent Filtering** 只过虑不发往内部地址X:x(地址：端口)的流量，而不考虑其流量的源地址和源端口。

- **Address – Dependent Filtering** 如果内部地址X:x没有先向外部地址Y发送流量的，NAT过虑外部地址Y发往内部地址X:x。即，只有X:x先发送流量到Y，Y才能发送流量到X:x。

- **Address and Port – Dependent Filtering** 如果内部地址X:x没有先向外部地址Y:y发送流量的，NAT过虑外部地址Y:y发往内部地址X:x。换句话讲，只有X:x先发送流量到Y:y，Y:y才能发送流量到X:x。

##基于STUN的NAT功能分类

NAT 主要分为两类一种是基本的NAT，另一种是NAPT (Network Address/Port Translator)。第一种NAT就是在地址转换过程当中只转换IP地址，而不会改变端口，而第二种会同时改变端口，在现目前的应用情况下，NAPT的设备最为常见。为了与其它资料兼容，在这篇文章当中，所有的NAT基本上代表的是NAPT。

STUN执行过程当中，按NAT对端口的映射方式将其分为四种类型。

### Full-cone NAT

这种NAT同时也称为one-to-one NAT。这种NAT的特点是一个内网的IP地址和端口被映射成一个外网的IP地址和端口(eAddr:ePort)。从内网中的iAddr:IPort端口所发送的任何数据都是都是通过这个外网固定eAddre:ePort发送出去的。同时任何外网的不同主机发送给这个NAT设备的eAddr:EPort的信息都将会转发到内网中iAddr:iPort当中去，如下图所示。

![full-cone](/img/ice/full-cone.png)

### (Address)-Restricted-cone NAT

和Full-cone NAT一样，一个内网的IP地址和端口被映射成一个外网的IP地址和端口(eAddr:ePort)。从内网中的iAddr:IPort端口所发送的任何数据都是都是通过这个外网固定eAddre:ePort发送出去的。但是对于外网的主机，只有内网发送的目标主机发送的数据才能通过NAT的这个端口。这个目标主机的任意端口都可以通过这个端口，如下图所示。

![Restricted-cone](/img/ice/Restricted-cone.png)

### Port-Restricted Cone NAT

和上面的两类非常相同，同样是端口一一映射。但是外网的主机只允许目标主机的目标端口的数据才能通过NAT到达内网的主机,如下图所示。

![port_nat](/img/ice/port_nat.png)

### Symmetric NAT

这种NAT与前面三种有非常大的不同，内网中的主机所发送的数据只要是不同的外网主机NAT都为它分配一个不同的端口，而内网的主机即使使用的同一个端口，只要发送的目标是不同的主机。那么NAT都会为之重要分配一个端口。而对于外网的主机，NAT只接受固定的外网主机的固定端口访问,如下图所示。

![symmetric](/img/ice/symmetric.png)

这一种类型的NAT在穿越的时候实际上几乎不可能的，不过幸好，由于这种NAT的实现比较复杂，所以在现实的应用当中比较少。

##STUN协议的工作原理

在网上的STUN协议都基本上出自2003年出版的RFC3489，全称为Simple Traversal of User Datagram Protocol(UDP) Through Network Address Translators(NATs)[4] 。但是在2008年RFC5389对STUN协议进行了一些改动，这个时候的STUN的英文全称是Session Traversal Utilities for NAT。原本RFC3489声称STUN是一个完全的穿越NAT的解决方案，而在新的RFC5389中，称原本的RFC3489只是穿越NAT的其中一种工具。新的STUN最大的改变就是增加了对TCP穿越的支持。

对于STUN的TCP穿越，现在还没有具体研究，但是基于先前的分析，可预测其结果也应该不会有太大的改变。

STUN的主要工作原理就是利用外网的计算机来辅助本地的计算机穿越NAT，得到本地计算机的外网IP地址和端口。STUN在工作过程当中需要服务器有两个公网的IP地址，或者两台公网上面的服务器。其实经过对NAT的分类，我们就已经发现了，只有前面三种NAT才有可能穿越。Symmetric
NAT不能够穿越。

为了更了的分析STUN的运行原理，图[STUN~T~HEROM]是一个STUN的解析图，在这个图当中，有两台客户计算机分别在NAT后面，在公网上面还有一个SERVER，它有两个IP地址。现在我们模拟STUN协议的运行过程。

- Client A的3321端口向服务器的IP1发送UDP数据包。Client A的数据包被NAT转换之后，换成公网的IP地址，同时端口变成了1233。这个时候NAT的公网IP向服务器IP1发送数据。服务器的IP1把接受到的数据中IP地址和端口打包封装成一个数据包，然后发回给Client A（这个时候是NAT的1233端口，NAT再转发给Client A）。正常情况下这个过程一定会执行成功，如果出现问题，那么多半是这个网络有问题，或者SERVER 并不存在。
- Client A收到了服务器IP1返回的自己在公网当中的IP地址和端口，将它与自己发送的IP地址和端口进行对比，如果发现这两个一样。那么就可以断定自己也在公网当中，这个时候看Client B的情况，如果Client B在内网当中，那么就可以使用反向连接方法。如果Client B也在公网当中，那么就可以直接连接。Client A发现自己的IP地址和端口和公网的不一样，那么就需要NAT穿越了。
- Client A知道自己在NAT之后了，这个时候就需要与服务器相互配合来确定自己的NAT类型。Client A先向服务器IP1发送一个请求，请求服务器使用IP2和不同的端口向Client A的公网地址和相同端口发送数据。如果Client A收到了服务器IP2的不同端口的数据，那么说明NAT是一个Full Cone NAT，这个时候就可以通知Client B进行连接，基本上NAT穿越成功了。不过遗憾的是在现实的应用当中Full Cone NAT实在非常少，所以一般情况下，很难接收到数据。如果Client A没有收到服务器IP2的数据，那么说明NAT有可能是Restricted Cone NAT也可能是Port Restricted Cone NAT，还有可能是Symmetric NAT。这个时候也需要做继续探测。
- Client A向服务器的IP2发送一个数据包，服务器IP2收到这个数据包之后，将Client A的公网IP和端口打包封装，并返回给Client A。在连接上面，这个过程一定能够成功。当Client A收到这个包之后，就与先前发往服务器IP1返回的包的端口进行对比。如果两个端口是不同的，那么可以确定这个NAT是Symmetric NAT，这个时候P2P直接不可能了，只能使用中继服务器来转发数据了。 如果两个端口是相同的，那么可以确定这个NAT，有可能是Restricted Cone NAT也可能是Port Restricted Cone NAT。
- Client A向服务器IP1发送一个请求，让服务器IP1以不同的端口像Client A的公网IP和端口回复一个数据。 如果Client A收到了回复数据包，那么可以说明这个NAT是Restricted Cone NAT。 如果没有收到，那么可以说明这个NAT是Port Restricted Cone NAT。 Client A和Client B一旦确定了自己的NAT类型和公网使用的IP地址和端口。那么Client A和Client B就可以直接向对方发送数据包，一旦发送了这样的数据包，那么它们就在各自的NAT上面打了一个洞，供对方的数据进来。

![all_frame_two](/img/ice/all_frame_two.png)
