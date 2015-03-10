---
layout: post
title: P2P客户端接口设计与实现
feature-img: "img/sample_feature_img.png"
---
##功能介绍
本文主要介绍P2P客户端的接口设计与实现，重点在于记录在整个接口设计过程中所遇到的问题，里面的内部有一些零散，对外没有太多的参考价值，仅当自己有需要的时候记录。
![协议嵌套](/img/p2p_sloution/p2p_sloution_framework.png)
<!--more-->

* 最顶层逻辑是User Contorl层，这一层创建下一层需要的部分，同时用户可以在这一层使用一个简单的函数使用程序运行起来，所有关于程 序的STUN地址的设置、Peer的连接都是属于其它程序通过网络客户端来进行连接的。
* 下一层主要分为三大部分，一部分是与服务器进行连接称为P2P Server Management，一部分是与本机的Socket进行服务器监听连接P2PProxyServerManagement，一部分是管理本机与代理Socket通信之间的管理，P2P Connection Management。其中，P2PConnectionManagement是整个程序当中最主要的部分，最重要的代理逻辑都在这一层当中实现。这三大部分全部使用的单例模式，也就是说，在整个程序的运行过程当中有且只有一个这样的对象。
* P2PConnectionManagement对象的主要功能分为两部分，一部分是管理P2PConnectionICE对象，一部分是管理ProxyP2PSession对象。对于P2PConnectionICE对象而言，P2PConnectionManagement主要是创建这个对象，同时调用P2PConnectionICE的ConnectionToRemotePeer函数来创建一个新的P2P连接，同时这个函数返回的一是一个可以使用的Stream对象，这个Stream对象就是程序以后过程当中代理数据交换的接口。一个新的Stream对象可以用来创建一个ProxyP2PSession对象。ProxyP2PSession就主要管理着具体的Proxy连接。
* 一个ProxyP2PSession当中主要有两种对象，一个是P2PConnectionImplementator对象，这个对象对应了一个Stream，它主要管理Stream对象的消息，同时对数据发送做加包头，对数据接收进行解包头的工作，这一层可以说是一个对Stream对象的封装。ProxyP2PSession另一个是一个ProxySocketBegin对象的Map结构体，一个peer连接当中可以复用多个数据通道，每一个数据通道都对应了一个ProxySocketBegin对象。在连接的过程当中，如果ProxyP2PSession管理的所有ProxySocketBegin对象都关闭的话，那么这个ProxyP2PSession对象也会立刻关闭。
* ProxySocketBegin对象只是由ProxyP2PSession对象来创建和最终消除的。
###ProxySocketBegin的生命周期

![协议嵌套](/img/p2p_sloution/ProxySocketBegin.png)

一个ProxySocketBegin对象管理着一个虚拟的Proxy连接,如图上图所示,在图的左边,有若干个客户端,这些客户端有的是RTSP客户端,也有的是HTTP客户端,每一个客户端连接Peer A之后,都会对应一个ProxySocketBegin对象来与之通信,而ProxySocketBegin对象发送的数据都会发送到一个P2PConnectionImpeletor对象中去,这个对象会对这些数据进行处理,然后通过同一个P2P通道发送出去,Peer B在接收到这些数据之后,会通过解包,把这些数据发送到相应的服务器中去.在一个结点当中,每一个ProxySocketBegin都对应有另一个结点的一个ProxySocketBegin的.这实际上是一个正常的网络逻辑,一台计算机建立的Socket,必须在另一台连接的计算机上有一个与之相对应的Socket连接起来的.

ProxySocketBegin的创建主要由两个地方完成，而ProxySocketBegin对象的消除只在一个地方进行。

* 由本地监听的服务器端口接收到一个请求之后，根据一个实际的Socket连接创建的ProxySocketBegin对象。
* 由远程Peer连接请求本地的Peer建立的一个新的Socket代理请求，而创建的ProxySocketBegin对象。

在最开始的时候,由ProxySocketServerFactory创建了一个服务器Socket,监听本地的一个端口,当一个新的连接被Accept之后,这个新产生的Socket就会形成一个ProxySocketBegin对象.新的Socket在ProxySocketBein当中与客户端进行通信，知道了客户端想要连接哪个服务器资源。这个时候ProxySocketBegin就会调用P2PConnectionManagement去查询这个服务器资源对应了哪个远端结点、如果这个远端结点已经连接了，那么P2PConnectionManagement就会直接把这个ProxyP2PSession对象返回给ProxySocketBegin,ProxySocketBegin得到Session对象之后，就把自己注册到这个Session当中，由这个Session来进行统一的管理。

如果P2PConnectionManagement查询到这个结点还没有进行连接，那么就会调用P2PConnectionICE的ConnectRemotePeer接口，让它建立一个新的Peer连接。ConnectRemotePeer会立刻返回一个Stream对象。P2PConnectionManagement就会使用这个对象创建一个新的ProxyP2PSession对象，然后返回去ProxySocketBegin对象。让它自己去注册。

在ProxySocketBegin调用Session的注册函数将自己注册到Session的同时，Session会检查当前是否已经建立了一个Peer连接，如果已经连接了P2P连接，那么就会直接通知这个ProxySocketBegin对象，当前已经建立起了P2P连接，你可以发送其它的数据了。如果没有还没有建立起连接，那么就会在连接建立好之后，通知所有Session下面的ProxySocketBegin对象当前P2P连接已经连接了。

ProxySocketBegin对象，在收到Session的P2P连接建立成功的消息之后，就会调用，Session的CreateClientSocket 或者 CreateServerSocket之类的函数，让Session向远程Peer结点发送一个建立对应ProxySocketBegin对象的消息。

##总体模块设计

第二章 的主要目标是介绍P2P Solution的总体类架构，从总体上介绍整个解决方案。首先从一个简单的主要框架图开始，然后逐渐细化，最后引出全局复杂的架构图。这个介绍的过程当中主要是阐明框架的特性和优缺点。不做太过细节的介绍。

这一节的主要目标是介绍在P2P Solution当中非常重要的关键流程，其中包括解包、拆包算法，Socket Number控制算法， Proxy Socket对数据的解析，资源释放和关闭等等。

##资源释放问题

产品当中，资源释放往往是一个非常大的问题，它常常会涉及到相当多的临界资源分配，还有异步、极端情况处理等等的问题，这一章的名称虽然叫做资源释放，但是实际上也可以看成“连接保障”，因为合理的释放和分配资源，才能保证一个程序能够长久的运行下去。这一章主要分为几个部分。

* 服务器资源释放和重连
* ICE部分的资源释放
* p2p connection management的资源连接和释放
* Proxy P2P Session 和 Proxy socket begin的资源释放
* Local Socket Factory的资源释放

这里面的每一部分的资源释放的同时，都会牵扯到其它部分的处理，实际上，以上的所有部分都没有一个独立的完整的过程，牵一发，而动全身。这一部分的合理性又对整个程序的架构是一个非常大的考验。

###服务器的资源释放

理论上来说，这一部分算是整个程序当中最为独立的部分了，在设计这一部分之初就考虑到了这个服务器非常有可能需要换，所以服务器和整个程序的关联并不大，它只需要维护几个接口不变就可以了。服务器的资源管理，主要是能够保证在服务器断开连接之后，能够自己重连。这种断开连接，不论是断开网络连接，还是其它的什么原因。不管怎么样，程序运行起来之后，要保证这个程序能够马上登录到服务器中去。

###ICE的资源释放


ICE部分在整个程序当中的主要功能是对对方的peer进行点对点连接，生成Stream对象，再根据Sream对象生成ProxyP2PSession对象，ProxyP2PSession对象最终会被P2PConnectionManagement管理着。所以，在ICE部分当中，除了自己本身初始化所需要的部分之外，并没有太多的资源需要管理的。
> 在上面的描述当中，我应该把ICE当中的ConnectRemotePeer的返回参数改一下，改成一个Stream对象，这样在调用这个函数的时候，我就不用把ProxySocketBegin对象传入到这个函数里面去了，这样可以让整个ICE部分更加的简洁，独立。

###P2PConnectionmanagement的资源释放

P2PConnectionManagement对象是一个全局唯一的静态对象，它在整个程序当中只会出现一个实例，如上面所说的，它的主要功能是管理ProxyP2PSession，和ICE部分，所有的客户端建立一个新的连接都必须要通过这个对象才能创建新的Session对象。每一个Session对象对应了一个Peer连接，除些之外，它还管理着P2P

##改进以及优化

这一章节的主要目标是提出对整个系统的改进，有一些改进是已经做过的，有一些改进是没有做的，但在未来可以做的。还有一些实用的功能扩展也可以在这个章节当中提出来。


1. 所有关于发送和接收数据的地方都必须使用连接保证，也就是如果发送数据不成功，需要把这个数据保存到，直到发送数据成功为止。在系统命令的发送过程当中，有可能只发送了一半的系统命令，这个时候，我就需要有一种机制来保证我接收到的一半的系统命令会自动删除。同时在源头上面，我还能重新发送这个系统命令。系统命令必须保证发送出去，而且必须保证一次性的发送出去。

2. 整个类的设计当中，我们遵守这样的原则，即：谁创建的对象，那么就由它自己消除这个对象。不能够越权处理，也不能够跨区管理。

###资源释放是程序设计的关键
正如做一件事情前80%的工作需要80%的精力，后20%却需要80%的精力一样。开始一件事情，可以凭借着一腔热血冲击一大段距离，但是到了最后一点的时候，往往因为自己疲惫了，不想再继续完成了。编程也是一样，前面写的时候，只考虑一般情况，这样写代码的感觉非常好，但是后面进行真正的边界检查的时候才会发现自己有各种各样的问题，只怪当时考虑得太简单的，没有留下接口。

所以，提前设计的关键就在于把一些比较边界条件处理的情况想好，把它们也融入到整个程序架构当中去。

##测试用例设计
***
* **参与测试程序**：两个Peer结点，一个RTSP客户端，一个RTSPServer。
* **测试描述**:两个Peer结点通过RTSP客户端正常进行连接，RTSPServer正常运行，然后RTSP客户端关闭。这个时候Peer需要关闭所有连接和通道，销毁ProxyP2PSession对象和ProxySocketBegin对象，关闭连接。
* **期望结果**：正常关闭。
* **实际结果**：正常
* **出现的问题**:刚刚开始的时候出现了ICE协议重复发送的问题，后面经过检查，原来是Session被及早的Destroy了导致ICE协议到来的时候没有找到相应的Session而引发的错误。Session及早 被Destory是因为我在接收到自己定义的P2P System Command的时候关闭的本地Tunnel,然后就会Destory Session。
* **解决方案**：我在接收到P2P System Command关闭通道的命令的时候，并不关闭通道，而等到ICE协议自动关闭通道。
* **其它可能的注意事项**：在单个连接的时候，有时候会出现连接不上的问题，这个时候Peer程序表现为一直在通过服务器和另一个Peer进行交互发送数据，但是连接不成功。
***

***
* **参与测试程序**：两个Peer结点，一个RTSP客户端，一个RTSPServer。
* **测试描述**:两个Peer结点通过RTSP客户端正常进行连接，RTSPServer正常运行，然后RTSP客户端关闭。这个时候Peer需要关闭所有连接和通道，销毁ProxyP2PSession对象和ProxySocketBegin对象，关闭连接，之后，RTSPServer再次请求peer建立一个新的连接，再次关闭，再次连接，再次关闭...
* **期望结果**：正常连接，关闭。
* **实际结果**：出错
* **出现的问题**:第一次连接关闭正常，第二次连接关闭正常，第三次连接关闭出现异常。这个时候可以看到RTSP Server上面有很多正在连接的资源。这种情况的出现，证明有一些资源释放并不彻底。
* **解决方案**：
* **其它可能的注意事项**：暂时没有
***

通过前面两个测试可以看出来，我的程序资源管理非常差，在资源释放的时候出现各种各样的问题。我必须要重新把ProxyP2PSession相关的资源全部检查一次再来测试。对于ProxyP2PSession来说，他下面主要管理两个部分，一部分是P2PConnectionImpletetor一部分是ProxySocketBegin对象。
除此之外，还有两个辅助的类，SendBuffer类，SocketTableManagement类。这些类的资源释放都要做好才可以。

对于P2PConnectionImpeletor来说，他由ProxyP2PSession创建的时候创建，
这个类在关闭的时候要注意要把与它关联的所有Signslot变量全部都断开连接。

###其它部分


通过这个过程，我渐渐的发现了一个这样的事实：不管是RTSP还是HTTP代理，其实我需要做的主要是一个，要实现HTTPServer这一部分和RTSPServer这一部分，因为对于p2p 代理的另一端来说，其实只需要连接需要连接的服务器而已，另一个Client的Socket并不需要对Client的数据进行任何特别的处理。所以这个地方需要改进。

* 当客户端第一次请求Peer连接服务器，但是客户端把自己的请求的服务器资源写错了，这个时候再把服务器资源改对，连接Peer会导致程序死掉。**已经解决**
* 在每一次连接的时候，服务器都会出现一次资源被请求了，然后释放了，再请求的信息。也就是说，可能在我建立连接的时候，第一次资源会没有连接上，第二次资源才连接成功。[贤哥说，这是正常现象。当一个客户端请求服务器一个资源的时候，服务器首先会去查询本地是否有这个资源，在这个查询过程当中就会显示这样的信息。服务器查询到这个信息之后，会通知客户端真的可以连接使用这个资源，所以就再次出现访问这个资源的现象。每一次客户端请求的时候，服务器都会检查这个资源，所以都会显示两次。]**已经解决**
* 多个客户端通过同一个Peer，连接另外一个Peer下面的服务器资源的时候，关闭一个客户端会导致程序出错。**已经解决**
* ProxySocketBegin当中的状态依然非常的混乱，必须要整理一下了。**已经解决**
* 深入研究一下智能指针，在一些类当中使用智能指针。
* 多个客户端（三个以上），通过同一个peer连接服务器，会出现有时候数据卡顿的现象，研究一下这个现象出现的原因。
* 加入Proxy连接30之后，关闭通道的功能。
* 使用线程来管理与服务器的连接，也使用线程来管理本地服务器注册的机制。

在Libjingle当中，资源的释放之所以如此的困难，主要的原因是因为这个库有大量的异步机制。有一些资源是被占用的，如果这个时候把这些占用的资源释放了，那么就会产生问题。所以，资源的释放必须使用异步的机制，不能够直接在一个函数当中调用delete之类的函数来直接释放这个资源。

对于P2PConnectionImpletetor所管理的Stream对象来说，他的资源释放时间应该是在Stream对象关闭的时候来释放自己。

除了P2P数据的接收、发送、关闭、Socket Block之外，其它的操作都使用异步的机制进行。

如果是客户端Socket主动断开的话，那么就先保证客户端的对象能够完全断开之后再释放其它资源[首先得保证安全删除自己的ProxySocketBegin对象]。如果是Stream对象主动关闭的话，那么就完全删除P2PConnectionImpeletor对象。在这一系列的情况下，状态的改变是一个非常重要的变量。

先断实体Socket连接，再断开P2P连接。

1. 对于P2P Socket关闭。
  1. 关闭internal socket.
  2. 使用Post消息，发送关闭P2P Socket成功的消息。
  3. 再使用Post消息，释放相关资源，最后Delete自己。

2. 对于internal socket主动关闭的情况。
  1. 关闭自己（internal socket close）。
  2. 发送关闭命令。
  3. 等待关闭命令成功的消息。
  4. 释放相关资源，最后Delete自己。

3. 关闭stream的时候需要注意，当proxy p2p session检查到本地已经没有了proxySocketBegin对象的时候，就应该等待30s之后才真正关闭通道。因为一个通道建立起来不容易，如果是因为出错的原因导致关闭的话，可以认为它会在短期内再次重新连接，这个时候这种机制就比较有用了。

###P2PConnectionImplementator的释放

对于P2PConnectionImplementator来说，它的生命周期和stream的生命周期是一样的，它的状态也主要是这个由stream来决定的。在资源释放的时候，有一个问题需要特别注意，就是stream关闭的时候，并没有马上关闭，它们还要进行ICE的协议来关闭，这一点就非常重要。P2PConnectionImplementator是一个独立的整体，不能够与ProxyP2PSession关联太多了。

P2PConnectionImplementator下面有两个重要的子过程，一个是SendDataBuffer,一个是DataMultiplexmachine,这两个都是在Stream存在的时候有用，在释放这两个的时候，必须要确保Stream已经关闭之后才能够进行。

###ProxySocketBegin当中的状态改变

p2psocket = SOCK NOT CONNECT
intsocket = SOCK CONNECTED

正常的客户端连接情况。

这个时候p2psocket就进行P2P连接，首先，它会和Internal socket 进行通信，客户端结点想连接的服务器资源。

1. 所以在读取Internal数据的时候，internal状态应该是SOCK CONNECTED,但是这仅仅是读取，并不发送，因为这个时候p2psocket还没有连接成功。
2. 读取到了服务器资源，这个时候就调用ProxySocketBegin对象的StartConnectBySourceIde进行P2P连接。这个时候p2p socket state的状态变成SOCK CONNECT PEER
3. 当与对方的P2P连接建立成功的时候，就会调用ProxySocketBegin的On P2P Peer Connect Succeed函数，这个函数会调用Proxy P2P Session的Create Client Socket Connect函数，创建一个P2P System Command，通知远端节点去连接真正的服务器。这个时候p2p socket的状态变成了SOCK PEER CONNECT SUCCEED.
4. 当远程实体Socket连接建立成功的时候，会回复一个给ProxySocketBegin对象的OnP2PSocketConnectionSucceed函数。在这个函数当中我们就可以把p2p socket state变成 SOCK CONNECTED状态了。
5. 其它方面：在p2p connect没有连接成功之前，理论上是不会出现关于P2P的消息的，所以OnP2PWrite,OnP2PRead,OnP2PClose都不会出现。同样的，在ReadP2PDataToBuffer当中，不应该写入数据。同样p2p socket没有建立连接成功，那么internal socket 也不应该发送数据。
6. 当p2p socket关闭的时候，当p2p socket关闭的时候，如上面所示的流程**把internal socket关闭，把internal socket改成SOCK CLOSE状态。**，发送一个执行SOCKET CLOSE的线程消息，在线程消息当中，把P2P Socket的状态改成SOCK CLOSE状态。
7. 当internal socket主动关闭的时候，首先把internal socket关闭，然后设置internal socket状态为SOCKET CLOSE.发送关闭p2p socket的消息。在接收On Other Side Socket Close Succeed函数当中接收到了对方关闭socket成功的消息，就把p2p socket的状态设置为SOCK CLOSE

正常的服务器连接情况

1. 当创建一个Client socket 的时候。初始化的状态当中，interal socket 的状态为SOCK CLOSE,而p2p socket的状态为SOCK PEER CONNECT SUCCEED.这个时候internal socket 对去连接相应的服务器。
2. 正常情况下，当服务器连接成功，会调用RTSPClientSocket的OninternalConnect函数当下就可以把p2p socket的状态改为SOCK CONNECTED，同时可以把internal socket的状态改成SOCKET CONNECTED.

##程序改进

1. 性能上面还需要提升，现在在我的电脑上面，开四个客户端，运行起来处理器占用高达15\%{}以上，在真的在现实场景当中，开起来，这个性能还不知道得成什么样子！！！
2. 所有对外的消息，都应该使用消息机制来传送，或者把传送消息放到一个函数的最后，像一个递归调用一样对待这种消息传送，不然，外部的机制太过于庞大，就非常有可能打乱内部的调用机制，这是一种非常危险的情况。
3. 在连接过程当中，在连接过程当中，要尽量使用智能指针，需要深入了解一下智能指针的实现以及用处。
4. 在设计的时候，对于每一层的设计应该考虑自己有能力能够独立释放
5. 线程消息机制是一种与传统的编程思维有很大不同的编程模式，有一点非常重要的是不要让线程所执行的消息运行太长的时间，这样会导致很多意想不到的问题，特别是对于一些网络应用来说，更要小心，对一些功能比较复杂的函数，在有必要的情况下，要拆分成若干个不同的子任务来分别执行。线程消息机制的本意就是，在使用的时候，不管是单线程还是多线程都要注意当前环境之下执行的异步问题。
6. 为不同的连接类型，开启不同的连接模式，对于RTSP这样的大数据流来说，使用不混合模式是比较好的连接，但是对于HTTP这样的数据量请求并不太大的情况，使用数据混合模式效果会更加的好。
7. 在proxy socket begin 和 p2p connection implementator 当中，接收和发送的张缓冲区应该一样大，而且设置成比较小的包，不应该太大，太大容易出问题。

> 模块化的更高要求，对于像libjingle这样的基于消息机制编译模式来说，在一个消息周期内，不应该存在太长久的消息处理，不然可能会造成其它的很多问题。
> 基于消息机制的编程模式更多的来说是一种编译的思维方式，与传统的编程方法有很大的不同，理论上我们也可以像传统的模式那样使用这样的机制，但是实际上，有很多细节性的问题必须要注意。不然必然会造成很多意想不到的问题。比如每一个消息的深度，就不应该太深。这是一个比较大的问题，值得我认真分析一下。

经过综合的考虑，我觉得整个p2p sloution的瓶颈非常有可能是来自于线程问题。不管怎么说，我的程序使用的依然是单线程，一个单线程总是有瓶颈的。

如果一个类一旦与线程相关，那么就必须在每一个类的函数当中加上对线程的判断，要把线程完全控制在自己的手里。

在写多线程的Socket连接的时候，要特别注意，不能够把一个事件（包括，读、写、关闭等等）执行业务逻辑太多。

##BUG以及解决方案
###ICE错误数据死循环问题}
**表现**：在关闭一个通道的时候，会产生大量的ICE数据，其中的数据内容都是标明ICE协议出现了错误，就是Session ID找不到的情况的错误。

**特征**：两边结点都会同时产生错误信息。其实是一个结点发到另一个结点，另一个结点又把这些数据发送过来，这样数据量就越来越大。

**出现的问题**：这种情况的出现是因为两边结点的Internal Socket都几乎同时关闭造成的。一边的关闭的时候会让ICE发送关闭通道的消息，另一个关闭的时候也会同时发送关闭通道的消息。但是正常的业务逻辑是一边发送了关闭通道的消息之后，就会把自己的通道销毁关闭，不再接收相应的ICE信息了。可是这个时候却接收到了ICE信息，就出现了相应的session id找不到的情况。

**解决方案**：让另一边internal 关闭的时候，并不发送ICE消息，意思就是并不真正的关闭ICE通道。

###同时关闭的问题
在进行P2P连接的时候常常出现两端同时关闭的情况，这种情况以前写程序的时候并没有考虑到，导致后面出现了非常多的问题。在后面对整个项目的代码进行梳理的时候，非常有必须对整个程序的业务流程再理一次，里面的BUG实在是太多了。

BUG的产生，主要还是因为业务逻辑混乱，代码非常混乱造成的。