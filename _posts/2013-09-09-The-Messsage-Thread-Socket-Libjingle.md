---
layout: post
title: WebRTC(Libjingle) 中的消息、线程与Socket
---

由于Message、Thread、Socket是三个密不可分的整体，单独分析其中的一部分会给人造成非常大的困扰，所以本章采用由浅入深的方式来分析这一块。

<!--more-->


整个章节分为三个层次：

1.  **总体描述三部分的整个架构和它们之间的关系。**这一部分主要描述的是这三大部分的架构图，使用图的方式来表示Libjingle的总体架构。这些架构图包括：线程与消息队列的关系，线程本身的管理方式，消息队列本身的管理方式，线程与Socket的管理方式，Socket本身的管理方式这五部分。最后给出五分部的最后总体架构图。

2.  **深入分析每一部分的具体实现和需要注意的部分，同时使用实例来验证相应的分析。**这一部分主要分析一些非常细节性的东西，依然分为三部分：分别是线程是如何识别自己的、消息机制和Socket机制是怎么样取出来的，消息队列分为了几种消息他们之间有什么联系，Thread当中有哪些接口可以让我们使用，Socket本身是怎么样继承管理的。

3.  **总结。**总结以文字为主，主要是对整个机制进行评价。它的重要性和效率。

libjingle整个软件的执行模式是基于消息的。在整个libjingle程序运行的过程当中，实际上可以看成是一个一个的线程在执行相应的消息。每一个线程都有自己的消息队列。同时libjingle库的主要功能是对网络的操作，所以每一个线程又管理着若干个套节字（socket）。这样下来，libjingle的主要功能就分为三部分。

-   对于线程本身的操作，主要包括创建线程，执行线程，中止线程，睡眠等等。

-   对于消息队列的操作，向自己的消息队列发送消息，向别的线程的消息队列当中发送消息，发送普通消息，发送有执行时间点的消息，取出一个消息并执行它。

-   对于Socket的操作，主要是得到一个创建Socket的工厂（SocketFatory）对象，然后通过这个对象创建相关的套节字。
![线程的重要组成部分](/img/message_thread/message_queue.png)

上图显示的是线程和消息、套节字之间的整体架构。可以看出，实际上Thread是一个面对程序员的对象，而MessageQueue是一个隐藏的对象。Socket的管理完全是由SocketFatory来管理的。

上面我们说到，线程的整个生命周期当中，实际上是在执行一个一个的消息。那么这些消息又是怎样的消息呢？它们以怎么样的形式来表示的，我们又应该如何使用？接下来，我就主要来回答这个问题。

通过上图我们可以看到，实际上消息本身是由MessageQueue这个对象来管理的。在它里面的一个链表消息队列和一个优先消息队列就存储了需要执行的消息。这个队列里面的格式呢？由如下代码可以看到，普通的消息直接是一个List结构，而优先队列当中的消息是一个封装了两次的STL优先队列。
{% highlight c++ linenos %}
//这是MessageList的定义
typedef std::list<Message> MessageList;

// 这是PrioityQueue的定义，它分为两部分，一部分是单个DelayedMessage消息，另一部分是一个
// STL的优先队列组成的队列。
// DelayedMessage goes into a priority queue, sorted by trigger time.  Messages
// with the same trigger time are processed in num_ (FIFO) order.

class DelayedMessage {
 public:
  DelayedMessage(int delay, uint32 trigger, uint32 num, const Message& msg)
  : cmsDelay_(delay), msTrigger_(trigger), num_(num), msg_(msg) { }

  bool operator< (const DelayedMessage& dmsg) const {
    return (dmsg.msTrigger_ < msTrigger_)
           || ((dmsg.msTrigger_ == msTrigger_) && (dmsg.num_ < num_));
  }

  int cmsDelay_;  // for debugging
  uint32 msTrigger_;
  uint32 num_;
  Message msg_;
};

class PriorityQueue : public std::priority_queue<DelayedMessage> {
 public:
  container_type& container() { return c; }
  void reheap() { make_heap(c.begin(), c.end(), comp); }
};

{% endhighlight %}

在上面的代码当中，不管是普通消息队列，还是优先消息队列，它们都有一个成员变量：Message结构。这个Message结构实际上就是线程执行消息的时候所要处理的信息。
{% highlight c++ linenos %}
struct Message {
  Message() {
    memset(this, 0, sizeof(*this));
  }
  inline bool Match(MessageHandler* handler, uint32 id) const {
    return (handler == NULL || handler == phandler)
           && (id == MQID_ANY || id == message_id);
  }
  MessageHandler *phandler;
  uint32 message_id;
  MessageData *pdata;
  uint32 ts_sensitive;
};
{% endhighlight %}

代码上面的代码显示的消息的具体格式，这种消息定义方式是一种比较常见的方法，在一般的基于消息机制的系统当中比如windows当中，消息的基本格式差不多都这样。在这个结构体当中有四个参数，这四个参数就是我们在传送消息的时候所需要指定的参数。

**message_id**
是消息的ID值，这个非常好理解，是用户自定义的消息类型，毕竟只有一个OnMessage回调函数，像windows消息中的机制一样，有可能是WM_PAIN或者WM_BUTTON类型的消息。用户可以根据这个ID字段定义不同的处理方式。

**ts_sensitive**
是决定消息是否是一个有时间要求的消息，如果它为0，那么它就会使用DoDelayPost函数传送到MessageQueue的优先队列当中去，默认时间是150ms，如果不为0，那么它就会传送到MessageQueue中的普通链表中去。实际上ts\_sensitive的有效参数只是0和非0的差别，Libjingle对这些消息使用的都是默认超时时间150ms，用户不能自己设置。在这里需要特别注意，是否传送到优先队列，只是取决于你是用DoDelay函数传递的消息还是用Post函数传递的消息，只是当你使用DoDelay传送一个消息，而ts\_sensitive设置的是0，那么这个消息将会以最快的速度得到响应。

**phandler**
是一个MessageHandler的参数。这个参数就决定了这个消息到底是传给哪个类，由哪一个类得到这个消息，这是整个消息结构体中最重要的一部分，它决定了消息机制的可扩展性。

**pdata**
的设计就比较巧妙了，实际上他只是一个MessageData类的指针，MessageData是一个基类，MessageData允许用户定制自己的消息。如代码[code:UDM]
所示。可以看到这个结构中除了构造函数和析构函数，就没有任何东西，连数据都没有，如果用户想要自己定制自己的消息数据，那就可以继承这个类，在子类当中加入新的数据。在OnMessage接收到MessageData指针之后，再转换成子类，自己的指针。
{% highlight c++ linenos %}
class MessageData {
 public:
  MessageData() {}
  virtual ~MessageData() {}
};
{% endhighlight %}
在上面的代码当中`MessageHadler *phandler`和`MessageDdata *pdata`一样都是一个没有实质内部的类，`MessageHandler` 是一个纯虚类。代码下面的代码是 `MessageHandler` 的定义，可以看到里面的一个纯虚的函数 `OnMessage`。如果我们的类想要接收到消息的话，那么我们只需要继承 `MessageHandler` 并实现 `OnMessage` 函数，那么我们这个类就可以接收消息了。而接收到的消息就是 `OnMessage` 的参数 `Message`。如上面所示。
{% highlight c++ linenos %}
class MessageHandler {
 public:
  virtual void OnMessage(Message* msg) = 0;

 protected:
  MessageHandler() {}
  virtual ~MessageHandler();
};
{% endhighlight %}

有了对于消息整体结构的认识，接下来我们就必须要知道如何发送一个消息。前面我们已经说过，线程继承自 `MesssageQueue`，而整个消息的运转对用户来说是透明的，我们只需要知道如何发关消息和如何接收消息就可以了。对于接收消息，上面已经说了，只要我们实现的类继承了 `MessageHandler` 就可以接收到消息，那么谁来发送消息呢。

当然是线程，前面已经说过，线程的整个生命周期内做的事情都是处理消息，那么当然，发送消息也需要有线程来执行。我们只需要一个线程的句柄就可以完成消息的发送了。在线程当中，发送消息有五个函数。
{% highlight c++ linenos %}
//在MessageQueue类当中定义
//public:
  virtual void Post(MessageHandler *phandler, uint32 id = 0,
                    MessageData *pdata = NULL, bool time_sensitive = false);
  virtual void PostDelayed(int cmsDelay, MessageHandler *phandler,
                           uint32 id = 0, MessageData *pdata = NULL) {
    return DoDelayPost(cmsDelay, TimeAfter(cmsDelay), phandler, id, pdata);
  }
  virtual void PostAt(uint32 tstamp, MessageHandler *phandler,
                      uint32 id = 0, MessageData *pdata = NULL) {
    return DoDelayPost(TimeUntil(tstamp), tstamp, phandler, id, pdata);
  }
//private:
  void DoDelayPost(int cmsDelay, uint32 tstamp, MessageHandler *phandler,
                   uint32 id, MessageData* pdata);
////////////////////////////////////////////////////////////////////////////
//在线程类当中定义
//public
virtual void Send(MessageHandler *phandler, uint32 id = 0,
    MessageData *pdata = NULL);
{% endhighlight %}
上面的代码显示的是这五个函数，其中前四个函数是在 `MessageQueue` 当中定义，后面一个Send函数是在线程当中定义的。`Send` 所发送的消息比较特殊，留在后面再来谈论。

从上面几个函数的定义，是不是发现它们的参数和前面所定义的消息的格式中的成员变量是一样的呢。所以不管这个发送消息使用了什么样的默认值，他们最终所要发送的消息都是代码前面消息格式末端的当中所示的那样的结构体。

- **Post**          将消息直接发送到当前正要执行的消息当中（普通链表）
- **DoDelayPost**   将消息直接发送到延迟消息队列当中（按时间排序的优先队列）
- **PostAt**        调用的DoDelayPost函数，可设定在某个具体的时间执行这个消息
- **PostDelyed**    调用的DoDelayPost函数，可设定在多少时间之后执行这个消息

上面列出了线程当中主要的发送消息的函数。我们主要用前面三个函数。这三个函数当中，我们主要使用的是 **Post** 和 **PostDelyed** 两个函数，因为这两个函数一个就可以把消息传送到普通消息队列当中，一个可以把消息传送到优先消息队列当中。

前面信息量有一点大，废话不多说，直接一个例子来演示如何使用这一套机制。

在这个示例当中，我将会使用线程来执行一系列的输出信息。有两个线程，一个主线程（程序从main开始执行的时候默认的线程），一个工作线程（自己创建的线程）。他们两个线程轮流的以出需要执行的消息，并输出一个字符串。程序当中，由于是多线程执行的，但没有使用资源互斥的机制，所以每一次输出的消息都在200ms之后才开始执行。
{% highlight c++ linenos %}
    #include <iostream>

    #include "talk/base/thread.h"

    //some basic define
    const int MAIN_THREAD_MESSAGE   = 0;
    const int WORKER_THREAD_MESSAGE = 1;
    const int MAX_MESSAGE_COUNT     = 10;

    //create my own message data
    class MyMessageData : public talk_base::MessageData{
    public:
      MyMessageData(const std::string &msg_string)
        :msg_string_(msg_string){
      }

      const std::string GetMsgString() const {return msg_string_;}
      static int msg_count_;
    private:
      std::string msg_string_;
    };

    int MyMessageData::msg_count_ = 0;

    class MyThreadMessage : public talk_base::MessageHandler{
    public:

      MyThreadMessage(talk_base::Thread *main_thread,
        talk_base::Thread *worker_thread)
        : main_thread_(main_thread),worker_thread_(worker_thread){}

    private:
      void MessageByMainThread(){
        ASSERT(main_thread_ == talk_base::Thread::Current());
        std::string msg = "I'am a message, posted by worker_thread";

        PostMyMessage(worker_thread_,msg);
      }

      void MessageByWorkerThread(){
        ASSERT(worker_thread_ == talk_base::Thread::Current());
        std::string msg = "I'am a message, posted by main_thread";
        PostMyMessage(main_thread_,msg);
      }

      void PostMyMessage(talk_base::Thread *thread, const std::string &msg){
        MyMessageData *my_message_data = new MyMessageData(msg);
        my_message_data->msg_count_ ++;

        if(thread == main_thread_)
          thread->PostDelayed(100,this,MAIN_THREAD_MESSAGE,my_message_data);
        else
          thread->PostDelayed(100,this,WORKER_THREAD_MESSAGE,my_message_data);
      }

      //implement the MessageHandler pure virtual function
      virtual void OnMessage(talk_base::Message *msg){

        //First, getting the msg struct by msg->pdata;
          MyMessageData* params = 
            static_cast<MyMessageData*>(msg->pdata);

          if(params->msg_count_ > MAX_MESSAGE_COUNT){
            delete params;
            return ;
          }

          switch(msg->message_id){
          case MAIN_THREAD_MESSAGE:
            {
              ASSERT(main_thread_ == talk_base::Thread::Current());
              std::cout << "--------------MAIN_THREAD------------------" << std::endl;
              std::cout << params->GetMsgString() << std::endl;
              std::cout << "Msg Count is " << params->msg_count_ << std::endl;
              std::cout << "-------------------------------------------" << std::endl;
              MessageByMainThread();
              break;
            }
          case WORKER_THREAD_MESSAGE:
            {
              ASSERT(worker_thread_ == talk_base::Thread::Current());
              std::cout << "--------------WORKER_THREAD----------------" << std::endl;
              std::cout << params->GetMsgString() << std::endl;
              std::cout << "Msg Count is " << params->msg_count_ << std::endl;
              std::cout << "-------------------------------------------" << std::endl;
              MessageByWorkerThread();
              break;
            }
            //
            delete params;
          }
      }
    private:
      talk_base::Thread *main_thread_;
      talk_base::Thread *worker_thread_;
    };

    int main(void){
      //Get the current thread, it is a main thread
      talk_base::Thread *main_thread = talk_base::Thread::Current();

      //Create a new thread named worker thread
      talk_base::Thread *worker_thread = new talk_base::Thread();

      //Start the worker thread
      worker_thread->Start();

      //instantiate a MyThreadMessage object, who receive and run the thread message
      MyThreadMessage my_thread_message(main_thread,worker_thread);

      //post a message to MyThreadMessage object by worker thread
      std::string msg = "I'am a message, the printer is worker thread.";
      worker_thread->Post(&my_thread_message,WORKER_THREAD_MESSAGE,
        new MyMessageData(msg));

      //going to  infinite message loop 
      main_thread->Run();

      //Stop the worker thread, in fact this programm never rech here.
      worker_thread->Stop();

      return 0;
    }
{% endhighlight %}
上面的例子有非常多值得注意的细节。从主函数开始，首先做的是创建了两个线程句柄，之所以叫线程句柄是因为new
一个线程对象并没有马上创建一个真正的线程。对于main\_thread来说，它只是一个线程对象的指针，通过talk~b~ase::Thread::Current()得到了当前线程的句柄，当前线程是主线程，在程序一开始的时候默认执行的线程。主线程的生命周期就是整个程序的生命周期，一个主线程线束了，那么也代表整个程序就结束了。所以在代码的到数第三行调用了线程类的Run函数。这个函数是让当前线程进行到一个无线的消息循环当中，开始执行自己消息队列里面的消息。

worker\_thread直接new了一个线程对象。但是这个时候并没有马上创建一个线程，当我们调用它的成员函数start的时候。才开始真正的创建了一个线程，当start函数调用之后，这个线程也进入了一个无限的消息循环当中，执行自己的消息。

所以，每一个线程创建最终都是要进入自己的消息循环当中去，正好印证了上面所说的，一个线程的生命周期当中，主要的任务就是执行自己的消息。

以上的分析只是一个非常简单的分析，应该说线程的主要功能和使用方法在上面的代码当中已经演示了。当然libjingle的线程消息机制隐藏在后面还有非常多值得注意的细节，在后面的部分我们会再次深入的分析其中的代码。

##深入分析线程消息机制
--------------------

通过上面的分析，我们已经对整个线程消息机制有了初步的认识，这一章的主要目的是深入分析各个部分的具体实现，并且辅助以各种示例来全方面剖析整个线程消息机制。

首先我们会先从线程开始，从线程如何管理，线程当中的一些重要的函数。
然后就是消息的管理，消息队列是如何管理的，消息是怎么样被取出来的，如何发送消息，如何执行一个消息。
接下来就是非常重要的重点Socket的管理，Socket是怎么样管理的，Socket又是如何使用的等等。

这一部分的内容会深入到libjingle线程消息机制的核心实现去，而且有非常多的细节性问题会分析到。面向的是那些想要知道libjingle如何实现线程的同志们，还有那些想把libjingle当中的消息机制加入到自己的项目当中去的同志。libjingle当中有非常多实用的库，代码高效美观，看得会让很多人心动，但可惜的是，除了极少部分的库之外，大部分的库都和libjingle的线程消息机制结合得非常紧密。libjingle的线程消息机制可以说是一种不同的程序设计思维，当你真正把libjingle当中的库应用到自己项目当中去的时候就会发现，它有一些地方总会和你的项目设计方法格格不入，这个时候你会怎么办？除了放弃之外，接下来就是把这个过程改了，不管是改成和你的项目相似的设计、还是把你的项目全部改成和libjingle的线程消息机制相似，这第一步一定是深入了解Libjingle的线程消息机制是怎么实现的，如果你对这个不感兴趣，接下来的内容，你可以只看一些示例就可以了，其它随便浏览一下，总有好处的。

### 线程的管理

通过上面的分析和代码示例，我们可以清楚的明白，线程和它的消息队列是一个比较统一而且独立的整体。我们可以把线程简单的看成是一个一个独立的运算器，当我们在使用它的时候，只要需要这个线程的句柄然后就可以做一系列的操作了。如果有多个线程，对于我们程序员来说也可以使用一定的机制来管理这些线程，保存它们，使用它们，或者做一个线程池等等之类的更加复杂或者简单的应用，来充分利用线程的功能。

固然，我们自己来管理自己所创建的线程是一个非常自然的逻辑，那么是不是说libjingle就不用再管理已经创建的线程呢？答案当然是否定的，对于每一个线程来说，libjingle实际上都在一个地方保存了这些线程的句柄。它会对这些线程进行统一的管理，它管理的功能非常简单，只是记录一下。因为libjingle的线程管理只为用户提供一个有用的接口**Thread::Current()**，是的，libjingle有自己的线程管理类，而这个类最主要的功能仅仅是向用户提供一个**Thread::Current()**函数而已。

代码[code:THREAD~D~ATA~E~XAMPLE]当中我们可以清楚的看到Thread::Current()这个函数的用处，它仅仅只是让用户得到当前的线程句柄。请认真想一想，在没有其它提示的情况下，你如何得到当前所执行的线程的句柄呢？当在多线程环境下的时候，你如何知道当前在执行这一段代码的线程是哪个？

让我们仔细来分析线程管理是怎么实现的。

在libjingle当中，Thread的管理是通过ThreadManager类来实现的，ThreadManager类一般情况下用户不会用到这个类，它所提供的接口到被Thread所使用了，唯一一个对用户比较重要的CurrentThread函数，用户也可以通过调用Thread的Current函数间接调用。在使用使用线程的通过当中，我们实际上会常常使用到ThreadManager类当中的函数，但实际上都是通过Thread来间接调用的。所以给我们的感觉好像没有它存在似的。

ThreadManger类在整个程序当中只有一个实例。libjingle使用一种非常简单的设计模式来完成这个功能：“单例模式(Singleton
pattern)”。在这里，不得不再说一句，libjingle当中使用的设计模式非常多，而且非常优秀，我分析这个库分析到现在，越来越觉得这libjingle的设计真心强大，Google的工程实在是太棒了。
{% highlight c++ linenos %}
    //ThreadManager的声明
    class ThreadManager {
     public:
      ThreadManager();
      ~ThreadManager();
      static ThreadManager* Instance();
      Thread* CurrentThread();
      void SetCurrentThread(Thread* thread);
      Thread *WrapCurrentThread();
      void UnwrapCurrentThread();

     private:
    #ifdef POSIX
      pthread_key_t key_;
    #endif

    #ifdef WIN32
      DWORD key_;
    #endif
    };
    //一些重要函数的实现

    // Use these to declare and define a static local variable (static T;) so that
    // it is leaked so that its destructors are not called at exit.
    #define LIBJINGLE_DEFINE_STATIC_LOCAL(type, name, arguments) \
      static type& name = *new type arguments


    ThreadManager* ThreadManager::Instance() {
      LIBJINGLE_DEFINE_STATIC_LOCAL(ThreadManager, thread_manager, ());
      return &thread_manager;
    }

    ThreadManager::ThreadManager() {
      key_ = TlsAlloc();
    #ifndef NO_MAIN_THREAD_WRAPPING
      WrapCurrentThread();
    #endif
    }

    ThreadManager::~ThreadManager() {
      UnwrapCurrentThread();
      TlsFree(key_);
    }

    Thread *ThreadManager::CurrentThread() {
      return static_cast<Thread *>(TlsGetValue(key_));
    }

    void ThreadManager::SetCurrentThread(Thread *thread) {
      TlsSetValue(key_, thread);
    }
    #endif

    Thread *ThreadManager::WrapCurrentThread() {
      Thread* result = CurrentThread();
      if (NULL == result) {
        result = new Thread();
        result->WrapCurrentWithThreadManager(this);
      }
      return result;
    }

    void ThreadManager::UnwrapCurrentThread() {
      Thread* t = CurrentThread();
      if (t && !(t->IsOwned())) {
        t->UnwrapCurrent();
        delete t;
      }
    }

{% endhighlight %}
上面的代码是libjingle当中ThreadManager的主要定义和实现，在这里我省略了一部分内容，不过不会影响到最终的结果。单模式的实现和原理具体细节就可去网上找，资料非常多。在这里我只是简单的描述一下。

单例模式的主要目标是保证全局只有一个实例，常常用在一些比较稀缺的资源或者像ThreadManager这样的管理类上面。要保证全局只有一个对象，在C++当中一般会使用一个静态的Instance创建函数，和一个静态的指向这个对象的指针变量来完成这样的功能，这个静态指针初始化为空。当我们想要得到这个对象的指针，就可以调用Instance这个静态函数，在这个函数当中，首先会判断是否静态对象指针已经指向了一个对象不为空了。如果为空，那么就创建一个对象，然后返回这个静态指针，我们用户就可以通过这个指针做一系列的操作。当用户在第二个地方调用Instance函数的时候，他会直接返回静态指针所指向的对象，从而保证全局只有一个对象。而我们用户不允许在其它任何地方使用new来创建一个新的对象。为了保证这种机制，常常把这个对象的构造函数声明成一个私有的构造函数，这样我们只能够通过Instance得到对象指针，而不能够创建新的对象。

说这么多有点糊涂了，具体我们可以看看上面的代码是怎么实现的。

首先我们要以看到ThreadManager声明当中有一个静态的函数Instance，这个函数就是我们上面所说函数。在这个函数当中有一个宏LIBJINGLE_DEFINE_STATIC_LOCAL，上面已经列出了它的实现，可以看出来，实际上这个宏就是定义了一个静态的变量，然后实例化一个ThreadManager对象。由于他是静态的变量，所以在第一次Instance函数调用的时会被赋值实例化一次，以后Instance函数调用的时候就不会再赋值了，这样也保证了全局唯一的对象。但ThreadManager的构造函数并没有声明为一个私有函数，这也许是设计上故意的实现，或者有其它的考虑，也有可能算一个小BUG吧。

在介绍ThreadManager后面几个函数的之前，我们必须要了解一种叫做Tls变量的技术。以下内容来自网络上的引用:

> 进程中的全局变量与函数内定义的静态(static)变量，是各个线程都可以访问的共享变量。在一个线程修改的内存内容，对所有线程都生效。这是一个优点也是一个缺点。说它是优点，线程的数据交换变得非常快捷。说它是缺点，一个线程死掉了，其它线程也性命不保;
> 多个线程访问共享数据，需要昂贵的同步开销，也容易造成同步相关的BUG.
> 如果需要在一个线程内部的各个函数调用都能访问、但其它线程不能访问的变量（被称为static memory local to a thread 线程局部静态变量），就需要新的机制来实现。这就是TLS。

线程局部存储在不同的平台有不同的实现，可移植性不太好。幸好要实现线程局部存储并不难，最简单的办法就是建立一个全局表，通过当前线程ID去查询相应的数据，因为各个线程的ID不同，查到的数据自然也不同了。大多数平台都提供了线程局部存储的方法，无需要我们自己去实现：
它主要是为了避免多个线程同时访存同一全局变量或者静态变量时所导致的冲突，尤其是多个线程同时需要修改这一变量时。为了解决这个问题，我们可以通过TLS机制，为每一个使用该全局变量的线程都提供一个变量值的副本，每一个线程均可以独立地改变自己的副本，而不会和其它线程的副本冲突。从线程的角度看，就好像每一个线程都完全拥有该变量。而从全局变量的角度上来看，就好像一个全局变量被克隆成了多份副本，而每一份副本都可以被一个线程独立地改变。

**分类：动态TLS和静态TLS**

用途：动态TLS和静态TLS这两项技术在创建DLL的时候更加有用，这是因为DLL通常并不知道它们被链接到的应用程序的结构是什么样的。

1．如果应用程序高度依赖全局变量或静态变量，那么TLS可以成为我们的救生符。因而最好在开发中最大限度地减少对此类变量的使用，更多的依赖于自动变量（栈上的变量）和通过函数参数传入的数据，因为栈上的变量始终都是与某个特定的线程相关联的。如果不使用此类变量，那么就可以避免使用TLS。
2．但是在编写应用程序时，我们一般都知道自己要创建多少线程，自己会如何使用这些线程，然后我们就可以设计一些替代方案来为每个线程关联数据，或者设计得好一点的话，可以使用基于栈的方法（局部变量）来为每个线程关联数据。

1 动态TLS

系统中每个进程都有一组正在使用标志（in-use flags），每个标志可以被设为FREE或INUSE，表示该TLS元素是否正在被使用。

进程中的线程是通过使用一个数组来保存与线程相关联的数据的，这个数组由TLS~M~INIMUM~A~VAILABLE个元素组成，在WINNT.H文件中该值被定义为64个。也就是说当线程创建时，系统给每一个线程分配了一个数组，这个数组共有TLS~M~INIMUM~A~VAILABLE个元素，并且将这个数组的各个元素初始化为0，之后系统把这个数组与新创建的线程关联起来。每一个线程中都有它自己的数组，数组中的每一个元素都能保存一个32位的值。在使用这个数组前首先要判定，数组中哪个元素可以使用，这将使用函数TlsAlloc来判断。函数TlsAlloc判断数组中一个元素可用后，就把这个元素分配给调用的线程，并保留给调用线程。要为数组中的某个元素赋值可以使用函数TlsSetValue，要得到某个元素的值可以使用TlsGetValue。

一般通过调用一组4个API函数来使用动态TLS：TlsAlloc、TlsSetValue、TlsGetValue和TlsFree。
**TODO:对于ThreadManger的理解是错的，需要改一下。**

知道了这四个函数，剩下的事情就简单了。回到代码[code:TMDDY]中，我们可以看到，除了单例模式之外，剩下的几个函数都与Tls对应的函数相关了。其中ThreadManage对应的构造函数和析构函数分别对应了Tls的TlsAlloc和TlsFree函数。接下来CurrentThread和SetCurrentThread函数分别对应了Tls的TlsGetValue和TlsSetValue函数。其它剩下的函数都直接或者间接的对这两个函数进行调用。

不难想像，有了Tls这样的特性，我们就能够非常合理的逻辑来设计一种机制知道当前所执行的线程是哪个线程了，如何得到当前运行线程的句柄。在Libjingle当中，它在当前线程的Tls数组当中存放了当前线程的指针。当我们想知道当前所运行的是哪个线程的时候，就可以调用CurrentThread来得到当前线程的Tls中所设置的值。从而得到当前线程的句柄。

在这里还有一个技巧的问题。在代码[code:THREAD~D~ATA~E~XAMPLE]中，我们在主函数main开始的时候得到了一个当前正在执行的线程signal_thread，我也常常使用线程，但是基本上没有刻意的得到当前主函数所执行的线程。看看[code:THREAD~D~ATA~E~XAMPLE]的第100行代码，它调用的是Thread的::Current()函数来执行的线程。上面分析了ThreadManager是一个单例模型，可以预见，在这个函数当中我们首先会创建一个ThreadManager类，然后做什么呢？

[code:CT]显示的是Current 函数调用的路径。
{% highlight c++ linenos %}
    // static
    // first
    Thread* Thread::Current() {
      return ThreadManager::Instance()->CurrentThread();
    }

    // second
    ThreadManager* ThreadManager::Instance() {
      LIBJINGLE_DEFINE_STATIC_LOCAL(ThreadManager, thread_manager, ());
      return &thread_manager;
    }

    //
    ThreadManager::ThreadManager() {
      key_ = TlsAlloc();
    #ifndef NO_MAIN_THREAD_WRAPPING
      WrapCurrentThread();
    #endif
    }

    //
    bool Thread::WrapCurrentWithThreadManager(ThreadManager* thread_manager) {
      if (started_)
        return false;
    #if defined(WIN32)
      // We explicitly ask for no rights other than synchronization.
      // This gives us the best chance of succeeding.
      thread_ = OpenThread(SYNCHRONIZE, FALSE, GetCurrentThreadId());
      if (!thread_) {
        LOG_GLE(LS_ERROR) << "Unable to get handle to thread.";
        return false;
      }
      thread_id_ = GetCurrentThreadId();
    #elif defined(POSIX)
      thread_ = pthread_self();
    #endif
      owned_ = false;
      started_ = true;
      thread_manager->SetCurrentThread(this);
      return true;
    }

    //
    void ThreadManager::SetCurrentThread(Thread *thread) {
      TlsSetValue(key_, thread);
    }

    //
    Thread *ThreadManager::WrapCurrentThread() {
      Thread* result = CurrentThread();
      if (NULL == result) {
        result = new Thread();
        result->WrapCurrentWithThreadManager(this);
      }
      return result;
    }


    //Third
    Thread *ThreadManager::CurrentThread() {
      return static_cast<Thread *>(TlsGetValue(key_));
    }

{% endhighlight %}
### MessageQueue的管理

其实MessageQueue和线程的管理是一样的，对于MessageQueue管理的是一个MessageQueueManager类。这个类也是使用的单例模型。MessageQueue对我们一般的用户来说更不会用到。这个类的主要功能是清除某发往某一个类的所有消息。这一点非常的重要。我们知道，Libjingle是基于线程的，当多个线程在运行的时候，消息机制运转起来，假如我们需要释放某一个类的对象，如果我们直接就delete了，那么在一些特殊的情况下，有一些消息本来是发往这个类的，但是在我们delete这个类的对象的时候这个消息还没有被执行，这个时候，当消息来了，那么就会生产致命的程序错误。

所在，特别需要注意，在释放一个能够接受消息的类的时候，必须要首先清除发往这个对象的所有消息队列中的消息才能够删除这个对象，否则后果就非常严重。

在这里，不得不多说一句，libinge的线程消息机制，是一种程序设计思维，在基于消息的程序当中，与传统的编程模型相比，它是有非常大的不同的。这种不同体现在很多方面，后面我们会开专门的一章来研究这个过程。

MessageQueue其实没有太多好说的，除了它的主要功能之外，其它的功能我们一般情况下是用不到的。如果读者对这个类有兴趣也可以自己去研究，因为这个类非常的简单。

### Socket

扯了半天，终于说到Socket与线程的关系来了。libjingle更多的是一个网络库，里面的核心部分在于处理与网络相关的事件，这一点非常重要，因为我研究libjingle的最初的目的就是为了使用libjingle的p2p库的。libjingle当中，每一个线程都可以管理几个socket，在这种基于消息的程序模型当中，这些socket的都使用的select的消息模型，在libjingle当中，虽然提供了阻塞式的socket接口，但是一般情况下是用不到的。

SOCKET在任意一个操作系统当中都会存在，基本上是一个通用的标准，一些对Socket的封装也主要是对socket的核心功能上面进行封装，更加符合相应的编程模型和设计方式。在libjingle当中，socket也被封装了多次，让我们来慢慢分析。

Socket首先会被Socket封装，在Socket类当中，主要封装了有关于Socket的相关函数，比如Bind、Connect、Send、Recv、Listen、Close等等。当然这些都只是一些虚函数，告诉各个继续类接下来应该实现什么样的接口。接下来Socket被AsyncSocet继承，这是一个单继承，从它的名字上可以看出这个类管理的是异步的Socket，从另一个侧面可以看出Libjingle实际上主要支持的是异步的Socket。

AsyncSocket实际上也是一个虚类，在里面没有实现任何实际性的内容。不过在这个地方出现了几个重要的变量Signal变量。如代码[code:ASYNC~S~OCKET]所示，在代码当中，可以看到有四个比较特别的变量。这是在libjingle当中大量使用的Signal
Slot机制，这种机制并不是Google原创，Google这是使用了一个开源的机制。
{% highlight c++ linenos %}
    // Provides the ability to perform socket I/O asynchronously.
    class AsyncSocket : public Socket {
     public:
      AsyncSocket();
      virtual ~AsyncSocket();

      virtual AsyncSocket* Accept(SocketAddress* paddr) = 0;

      // SignalReadEvent and SignalWriteEvent use multi_threaded_local to allow
      // access concurrently from different thread.
      // For example SignalReadEvent::connect will be called in AsyncUDPSocket ctor
      // but at the same time the SocketDispatcher maybe signaling the read event.
      // ready to read
      sigslot::signal1<AsyncSocket*,
                       sigslot::multi_threaded_local> SignalReadEvent;
      // ready to write
      sigslot::signal1<AsyncSocket*,
                       sigslot::multi_threaded_local> SignalWriteEvent;
      sigslot::signal1<AsyncSocket*> SignalConnectEvent;     // connected
      sigslot::signal2<AsyncSocket*, int> SignalCloseEvent;  // closed
    };

{% endhighlight %}

在Libjingle当中，消息、线程、套节字管理是三个不可分割又非常重要的核心部分，可以这样说，这部分是Libjingle的核心驱动引擎。Libjingle所有功能基本上都是建立在这个机制之上的，想要研究Libjingle的主要功能和原理就必须首先了解这三个部分。无论你想用Libjinlge来做什么，这一部分都是必须要理解的。在这一章我们主要来讨论关于Libjingle中的这三个部分，它们之间的联系、工作流程，以及如何使用和扩展它们。

Overview
--------

线程在libjingle当中由talk~b~ase::Thread这个类统一封装，对外的表现就是这样一个类。我们对于线程的所有操作，都是通过这个类来执行的。这个类当中已经封装了不同操作系统地层的差异，实际上这是一个非常优秀的线程库。

在libjingle当中，每一个线程都对应了一个**消息队列**和一个管理sock的**网络套节字队列**。如图[fig:TSTF]所示。不要惊讶，这只是一个非常非常简略的Thread、消息、套节字之间的关系图。接下来的分析当中，我们将会看到里面的全部细节。

我们将从线程说起，之后介绍消息机制，再之后分析网络管理部分。这三部分是不可分割的一个核心整体。

Thread and Message
------------------

线程和消息的管理在libjingle当中是不可分割的两个完整的整体，libjingle的整个程序架构是基于消息的。整个程序的主要运行方式就是执行一个一个的消息。而线程的存在也是为了使用不同的线程执行不同的消息，从而提高整个程序的效率。

消息的主要定义是在messagequeue.h当中的MessageQueue类里面,由名称就可以看出，对于消息来说，管理他们的是一个队列。在消息当中有两类消息，一类是平常的消息，先到先执行（代码表现为一个std::list），另一类是定时消息，只有当消息经过一段时间之后才执行的消息(代码表现为std::priority\_queue,不过具体实现当中继承了一次)。

消息对我们开发者来说非常的重要，不过libjingle将消息封装之后，我们只需要记住几点。

-   每一个消息队列都对应一个线程，反过来说也一样，每一个线程都有自己的消息队列。这一个线程的生命周期全部任务就是在执行消息队列里面的消息。

-   线程类（talk\_base::Thread）继承自消息类(talk\_base::MessageQueue)，对消息的所有操作都是通过线程类提供的接口来进行的。一般情况下，我们并不会直接调用MessageQueue里面接口。

这几点对于我们理解libjingle的消息机制非常重要。不过，从对于线程类的定义[CODE:DOT]就可以看出来。这是一个非常简单的事实，也是一个简单的原则，我之所以强调这两点，是因为这两点真的非常非常重要。后面我们就会看到了。
{% highlight c++ linenos %}
    class Thread : public MessageQueue {
      //...
    }
{% endhighlight %}
如果我们要使用libjingle的消息的话，比如向消息队列当中传送我们自己需要执行的消息，这一系列的接口实际上都是由Thread这个类提供的。我们只会通过Thread的接口来操作消息，而不会直接操作到消息类本身的接口。

实际上，在Thread当中，我们仅仅只需要有三类接口，一类是关于消息机制的接口，一类是关于Thread本身创建，运行的。还有一类是关于Socket的。

接下来，我们慢慢的分析整个消息机制。首先要分析的是MessageQueue的管理，然后是Thread的管理，再后面是他们两个的联系，最后是使用几个示例来说明如何使用这种机制。

### MessageQueue的管理

我们已经知道，每一个线程都有一个消息队列MessageQueue，而当一个线程结束的时候，总

在libjingle当中，管理Thread的是ThreadMenager类，这个类，在全局只存在唯一的一个，对于这个类，libjingle使用了设计模式当中的单例模式来进行定义的。

{% highlight c++ linenos %}
    class ThreadManager {
     public:
      ThreadManager();
      ~ThreadManager();
      static ThreadManager* Instance();
    };
{% endhighlight %}

在libjingle当中，线程类是继承自MessageQueen类的，所以这两个之间的联系非常的紧密，实际可以这样理解，线程的接口其实是MessageQueue的接口，在整个libjingle库中，我们想要对MessageQueun操作，

当我们一个程序开始执行的时候，其实这个时候就已经开始了一个线程，这个线程常常被称为主线程

在Libjingle实际的运行过程当中，一些程序的功能非常简单，可能并不需要多线程，比如在Libjingle中的STUN
server 示例和 TURN
Server示例都只是使用了一个线程。这个时候只需要一个Signaling
Thread就可以了。多线程的程序主要是为了程序的效率考虑，当处理大量网络数据的时候，当为了界面不阻塞的时候，多线程的设计就非常的有用了。

使用多线程的意义和时间我就不多说了，线程在libjingle当中，基本上是最为核心的部分，这一部分是所有部分的驱动引擎。这一部分直接关系着整个工程的编程风格和功能大小。

Message
-------

在libjingle当中，每一个线程都会有自己的一个消息队列，实际上在整个线程的生命周期内，线程的主要任务就是从自己的消息队列中取出消息然后执行。当然这里的消息包含了线程内部的消息，还有来自于网络的读写消息。

The Message In the Libjingle
----------------------------

消息机制，在Libjingle当中是非常重要的核心一部分，消息机制贯彻了Libjingle的方方面面。关于消息，一共有三个比较重要的类。

-   MessageQueueManager类，顾名思义这个类的主要功能是管理所有MessageQueue队列，在Libjingle当中，每一个线程都有一个自己的MessageQueue，而整个程序当中只有一个MessageQueueManager，它的功能实际上非常的简单，初始化、添加一个消息队列、删除一个消息队列。在整个程序当中，并没有太大的用处。但是它保证了消息机制的完整性。

-   MessageQueue类，这个类是消息机制当中最核心的类。它的主要功能包括发送消息、接收消息、取出消息。消息机制的大部分工作都在这个类当中实现。

-   Message类，它是消息的实体，所有的消息都以这样的结构体的形式体现。不过在实际使用过程当中，他被封装、继承了几次，会出现各种各样的变体，但最终的形式都是这样的。

以上三个类它们之间又有包含关系，MessageQueueManager包含了若干个MessageQueue，MessageQueue又包含了若干个Message结构体。如下图所示，其中MessageQueue当中有一个重要的Get函数，它用来取出消息。Get函数是主要用于被外界的函数所调用，外界函数通过Get得到一个消息之后，再进行处理。所以消息机制除了以上三个类之外，还需要有其它的方面来全程驱动。这些部分我们将在后面的过程当中慢慢讨论。
本节在上面的总体陈述之后，后面的内容主要分为四个重要部分。

-   第一个项目 详细描述MessageQueue的工作原理。

-   第一个项目描述Message结构体，以及消息的各种变体和MessageHandler的功能。

-   第一个项目描述MessageQueueManager如何管理全部的消息队列。

-   第一个项目举一个综合的实例，来实践整个消息机制。

![消息机制模型](/img/message_thread/get_message.png)

### MessageQueue

在Libjingle中，消息是由MessageQueue来管理的，MessageQueue当中定义了一个优先队列，一个普通的链表，这两个数据结构代表两种消息类型，都是用来存储从其它线程发送过来的消息。

实际上，这个优先队列可以看成是一个保存延迟消息的延迟队列，以时间排序。而普通链表可以看成当前正要执行的消息。

MessageQueue定义了取出消息的Get函数，每一次调用它，他就会返回一个可以执行的消息。Get函数会先查看在优先队列（延迟消息）当中是否有要处理的消息，如果优先队列当中有一个消息到达处理时间，就把这个消息从优先队列拿出来，加入到普通链表（当前正要执行的消息）的后面。然后再查看普通的链表当中是否有可以处理的消息，如果有就返回最前面的第一个，当这两个队列当中都没有可处理的消息之后，它就会去查看网络套节字是否有可以处理的事件发生，如果有就返回套节字的消息。MessageQueue的处理流程如图[MessageQueueIn]所示。

![MessaeQueue中的Get执行顺序](/img/message_thread/dispatcher_message.png)
[MessageQueueIn]

网络套节字的流程，在Message这一节不会深入讨论，后面有专门的部分会谈到网络套节字与消息队列之间的联系。

MessageQueue定义了传递消息的函数，传送有时延的消息使用DoDelayPost（也可以使用PostAt，PostDelayed这两个函数，不过他们的实现都是一样的，最终都是调用的DoDelayPost函数），他会把消息加入到优先队列当中，传送普通消息使用Post函数，想从消息队列当中得到一个可处理的消息就使用Get函数。具体内容如表[MessageType]所示。

[h]

  ------------- ----------------------------------------------------------- ---------
  Post          将消息直接发送到当前正要执行的消息当中（普通链表）          publick
  DoDelayPost   将消息直接发送到延迟消息队列当中（按时间排序的优先队列）    publick
  PostDelyed    调用的DoDelayPost函数，可设定在多少时间之后执行这个消息     publick
  PostAt        调用的DoDelayPost函数，可设定在某个具体的时间执行这个消息   private
  ------------- ----------------------------------------------------------- ---------

[MessageType]

### 消息机制的基本代码分析

Libjingle当中类与类之间的关系非常的紧密，这些类之间有着看似非常复杂，但是使用起来却给人非常简洁的特性，我们从他的消息定义当中就可以窥见一斑。我们首先看看一个纯虚类MessageHandler。
{% highlight c++ linenos %}
    class MessageHandler {
     public:
      virtual void OnMessage(Message* msg) = 0;

     protected:
      MessageHandler() {}
      virtual ~MessageHandler();

     private:
      DISALLOW_COPY_AND_ASSIGN(MessageHandler);
    };
{% endhighlight %}

MessageHandler是一个纯虚类，它要求想要接收消息的类都必须继续它，并实现其中的OnMessage函数。传给这个类的消息会自动调用OnMessage函数。可以看到OnMessage函数的参数是Message类型的参数。Message是一个结构体，它的定义如代码[DefineMessage]
所示:
{% highlight c++ linenos %}
    struct Message {
      Message() {
        memset(this, 0, sizeof(*this));
      }
      inline bool Match(MessageHandler* handler, uint32 id) const {
        return (handler == NULL || handler == phandler)
               && (id == MQID_ANY || id == message_id);
      }
      MessageHandler *phandler;
      uint32 message_id;
      MessageData *pdata;
      uint32 ts_sensitive;
    };
{% endhighlight %}

这个消息的结构体也是保存在MessageQueue中优先队列和普通链表当中的结构体。这个结构体实际上包括了很多信息。

**message\_id**
是消息的ID值，这个非常好理解，是用户自定义的消息类型，毕竟只有一个OnMessage回调函数，像windows消息中的机制一样，有可能是WM\_PAIN或者WM\_BUTTON类型的消息。用户可以根据这个ID字段定义不同的处理方式。

**ts\_sensitive**
是决定消息是否是一个有时间要求的消息，如果它为0，那么它就会使用DoDelayPost函数传送到MessageQueue的优先队列当中去，默认时间是150ms，如果不为0，那么它就会传送到MessageQueue中的普通链表中去。实际上ts\_sensitive的有效参数只是0和非0的差别，Libjingle对这些消息使用的都是默认超时时间150ms，用户不能自己设置。在这里需要特别注意，是否传送到优先队列，只是取决于你是用DoDelay函数传递的消息还是用Post函数传递的消息，只是当你使用DoDelay传送一个消息，而ts\_sensitive设置的是0，那么这个消息将会以最快的速度得到响应。

**phandler**
是一个MessageHandler的参数。这个参数就决定了这个消息到底是传给哪个类，由哪一个类得到这个消息，如果对这个机制不太理解可以看看本书后面的Appendix
the important knowledge of
C++。在最后的调用的时候，就是使用的phandler这个“句柄”来调用具体的类的OnMessage函数。

**pdata**
的设计就比较巧妙了，实际上他只是一个MessageData类的指针，MessageData是一个基类，MessageData允许用户定制自己的消息。如代码[UserDefineMessage]
所示。可以看到这个结构中除了构造函数和析构函数，就没有任何东西，连数据都没有，如果用户想要自己定制自己的消息数据，那就可以继承这个类，在子类当中加入新的数据。在OnMessage接收到MessageData指针之后，再转换成子类，自己的指针。
{% highlight c++ linenos %}
    class MessageData {
     public:
      MessageData() {}
      virtual ~MessageData() {}
    };
{% endhighlight %}

### MessageQueueManager

顾名思义，使用MessageQueueManager主要是用来管理MessageQueue的。实际上，Libjingle对MessageQueue的统一管理要求并不多。在MessageQueueManager当中只有三个比较有用的函数。Add，Remove，Clear。Add和Remove是加入或者删除生个消息队列。而Clear是消除所有消息队列中包含某个MessageHandler的消息，如代码[ClearFunction]所示。
{% highlight c++ linenos %}
    void MessageQueueManager::Clear(MessageHandler *handler) {
      ASSERT(!crit_.CurrentThreadIsOwner());  // See note above.
      CritScope cs(&crit_);
      std::vector<MessageQueue *>::iterator iter;
      for (iter = message_queues_.begin(); iter != message_queues_.end(); iter++)
        (*iter)->Clear(handler);
    }
{% endhighlight %}

关于Clear函数，具体来说，一个类只要继承了MessageHandler就可以在类中定义OnMessage函数，从而接收到相应的消息。但是当这个类的生命周期结束，内存被释放之后，所有发往这个类的消息都应该被清除，而MessageQueueManager就提供了这样的功能，这也是MessageQueueManager的主要功能。

除此之外，MessageQueueManager比较重要的一点是，他遵守了一个管理的模式，在下面将要介绍到的Thread当中，Thread
Manager也遵守了这样的一个模式。具体来说，在MessageQueueManager定义了一个静态成员变量instance\_和一个静态成员函数Instance()。他们之间最大的联系如代码[StaticValue]所示。
{% highlight c++ linenos %}
    MessageQueueManager* MessageQueueManager::instance_;

    MessageQueueManager* MessageQueueManager::Instance() {
      // Note: This is not thread safe, but it is first called before threads are
      // spawned.
      if (!instance_)
        instance_ = new MessageQueueManager;
      return instance_;
    }
{% endhighlight %}

可以看到，当在第一次调用Instance函数的时候，instance\_变量为NULL，所以就会创建一个MessageQueueManager对象。这是全局唯一的一个对象，在一个程序当中有且只有一个。

在一个MessageQueue中，当传递消息给这个队列的时候，他就会调用MessageQueue中的EnsureActive函数，而EnsureActive函数在第一次调用的时候就会把当前的MessageQueue加入到MessageQueueManager当中去，以后调用EnsureActive将不会做任何事情。

当一个MessageQueue要销毁的时候，在它的析构函数当中会把当前的MessageQueue移除MessageQueueManager。

这样的一个总体过程，就实现了管理MessageQueue的所有功能，控制力比较弱，但是非常有必要，特别在需要统一删除所有发往某一个MessageHandler的时候，这种管理机制最重要。

### 消息映射示例

说得太多，太过于抽象，在这里我们使用一个示例来演示以上的过程，以便加深理解。

**派生MessageData类，我们的消息只是一个整数。**
{% highlight c++ linenos %}
    struct TestMessage : public MessageData {
        explicit TestMessage(std::string v) : message_str(v) {}
        virtual ~TestMessage() {}
        //自定义数据
        std::string message_str;
    };
{% endhighlight %}

**派生MessageHandler 类实现自己的OnMessage函数 **
{% highlight c++ linenos %}
    class MyMessageData : public MessageHandler
    {
    public:
        MyMessageData(){};
        virtual void OnMessage(Message *pmsg) {
            TestMessage* msg = static_cast<TestMessage*>(pmsg->pdata);
            std::cout << "=========== Recive a message ===========" << std::endl;
            std::cout << "MESSAGE ID >>\t\t" << pmsg->message_id << std::endl;
            std::cout << "MESSAGE RICE TIME>>\t" << Time() << std::endl;
            std::cout << "MESSAGE STRING>>\t" << msg->message_str << std::endl;
            std::cout << "=========== End a message ===========" << std::endl;
            delete msg;
        }
    };
{% endhighlight %}

在这里我实现的是main\_thread\_message类，里面除了一个什么也没有的构造函数，就一个OnMessage函数。在OnMesage函数当中，首先把Message数据类转换成TestMessage类，Message是TestMessage的父类，所以在传递消息时候也可用TestMessage数据类。

**在主函数当中，发送并处理消息。**
{% highlight c++ linenos %}
    MyMessageData   test_message_client;
    MessageQueue    test_message_queue;
    uint32      now = Time();
    const uint32    ONE_SECOND = 1000;
    //传递消息
    //1. 传递实时消息
    for (int i = 0; i < 5; i++)
    {
        std::string post_message_str = "Current Message Type : Realy Message\t";
        test_message_queue.Post(&test_message_client, i, new TestMessage(post_message_str));
    }
        //2. 传递定时消息，设定在未来某个时间点执行
    for (int i = 0; i < 5; i++)
    {
        std::string post_message_str = "Current Message Type : Post At Time Message\t";
        test_message_queue.PostAt(now + i*ONE_SECOND, &test_message_client, i, new TestMessage(post_message_str));
    }
    //3. 传递定时等待消息，设定等待时间，等待多少时间之后执行
    for (int i = 0; i < 5; i++)
    {
        std::string post_message_str = "Current Message Type : Post At Time Message\t";
        test_message_queue.PostDelayed(i*ONE_SECOND, &test_message_client, i, new TestMessage(post_message_str));
    }
    Message get_message;
    //设定接收消息15s, 15s之后停止接收消息。
    uint32  wait_time = 15000;
    //接收消息，并发送消息到接收者中去
    while (test_message_queue.Get(&get_message,wait_time))
    {
        get_message.phandler->OnMessage(&get_message);
    }
{% endhighlight %}

实际上Libjingle的消息映射除上面所讲的部分，还有很多细节部分没有说到，不过这一点并不重要，包括上面的代码只是为了让我们能够更好的理解Message过程，一般情况下我们是用不到的，这些操作都已经被封装到内部了。这篇文章主要的目标是搞懂如何使用Libjingle的库，所以并没有太过深入的讨论具体的技术实现细节。