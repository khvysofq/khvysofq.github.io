---
layout: post
title: Task的设计与分析
---
#Task的设计与分析
Task是一个状态机，主要用于应用在单个线程里面执行多任务的情况。VZSIP中的Task是移植google libjingle里面的Task，并对它做了一些小的适应性修改。每一个Task都是一个小型的状态机，也就是说每一个Task都有自己的状态。这些状态之间的转移是通过推动TaskRunner来进行的。TaskRunner负责运行所有的Task，让Task从一个状态转移到另一个状态，直到Task结束为止。

![状态机的类关系](figure\\state_machine_class.png)

上图是整个状态机的类关系，一共有三个类。

* `TaskParent`，是所有类的基类。这个类做的是一个连接关系，典型的[中介者模式](http://blog.csdn.net/wuzhekai1985/article/details/6673603 "中介者模式")。它连接的是Task和TaskRunner两个类。
* `TaskRunner`，前面已经说过，TaskRunner是推动者，里面管理了当前需要执行的Task，同时要运行所有的Task都调用TaskRunner里面的函数来运行。所有的Task要运行的话，都必须把自己注册到TaskRunner里面，然后才能够由TaskRunner统一管理并且运行。
* `Task`，这是一个核心类。所有的具体任务都在这个类里面定义并且执行。每一个Task还可以有一系列的子Task，子Task还可以有子Task。这样就形成了一颗Task树。在树的最顶层就有一个Top Task。这样管理起来非常方便，我们只需要注意对这个顶层Task进行管理就行了。Task在设计的时候，同时考虑到了应用在多线程异步的情况，特别是可以与网络事件关联起来，用处非常大，后面的VZSIP里面就有关于Task与网络事件融合的情况。

所有的Task都被TaskRunner管理着，一个Task继承者可以是另一个Task的子类，也可以是TaskRunner本身。实际上，TaskRunner就是上面所说的Top Task。所有的Task都是他的Child，我们在管理的时候也只需要管理这个TaskRunner就可以控制整个业务。要启动一个Task则需要调用这个Task的`Start`方法，一旦调用了这个方法，就在`ProcessStart()`里面处理自己的业务。而我们在定义自己的Task的时候，只需要注意在什么状态下Task应该做什么样的工作这样的框架下面，专注于自己的业务，让一切变得非常简单。

下面，将会对Task进行比较深入的分析。首先每一个Task内部都有固定的几个重要的状态。
{% highlight c++ linenos %}
// Executes a sequence of steps
class Task : public TaskParent {
 public:
  Task(TaskParent *parent);
  virtual ~Task();
  // Omitted some content, see the code task.h
  void Start();
  void Step();
  // Omitted some content, see the code task.h 
protected:
  virtual int OnTimeout() {
    // by default, we are finished after timing out
    return STATE_DONE;
  }
 protected:
  virtual std::string GetStateName(int state) const;
  virtual int Process(int state);
  virtual void Stop();
  virtual int ProcessStart() = 0;
  virtual int ProcessResponse() { return STATE_DONE; }
  // Omitted some content, see the code task.h
};
{% endhighlight %}
{% highlight c++ linenos %}
  enum {
    STATE_BLOCKED = -1,
    STATE_INIT = 0,
    STATE_START = 1,
    STATE_DONE = 2,
    STATE_ERROR = 3,
    STATE_RESPONSE = 4,
    STATE_NEXT = 5,  // Subclasses which need more states start here and higher
  };
{% endhighlight %}
如上面的枚举变量所示，所有的Task都内置了这七种状态。
* `STATE_INIT`，表明当前的Task刚刚初始化完成，还没有进行任务工作，当声明一个Task的时候，就会默认在这个状态下面。
* `STATE_START` 当用户调用了一个Task的`Start()`之后，如果没有出错，就会进入这个状态。在这里面会回调用记为Task所实现的虚函数`ProcessStart()`，用户就可以在这里面初始化自己所想要的资源，并且做相应的任务工作。当用户在实现`ProcessStart()`函数的时候，他的返回值就直接状态了这个Task将进入什么样的状态中。
* `STATE_RESPONSE`，这个算是Task第二个正常状态，如上面Task类里面`ProcessResonse`函数的实现一样，如果用户的子类中没有重载这个函数，那么进入`STATE_RESPONSE`状态的Task会马上进入`STATE_DONE`的状态。
* `STATE_DONE`，当一个Task进入`STATE_DONE`之后，就表明这个Task已经执行完成，系统在接下来将会把这个Task删除。特别需要注意的一点是，当进入`STATE_DONE`之后，删除Task之前，父类是不会有什么方法通知子类要把这个Task删除的。所以，自己的重载Task的时候需要特别注意，严格执行整个Task的生命周期流程。特别是自己要清楚什么时候Task会进入`STATE_DONE`状态，注意在进入这个状态之前把自己相关的资源释放了。
* `STATE_NEXT`，在现实过程中，一个Task可能会有很多状态，所以在这种情况下，Task也提供了接口能够自己定义自己的状态，不过需要注意的是，要扩展自己的Task就需要子类继承Task的`virtual int Process(int state)`函数，在这个函数里面处理自己的状态。在这里面，如果得到的是Task的基本状态，那么还是需要再调用父类的`Process`函数，以更能够检索到前面的状态，供整个系统管理。
* `STATE_ERROR`，不管是在`ProcessStart`、`ProcessResponse`还是在自己所定义的状态处理函数里面，任何时候都可以返回`STATE_ERROR`，这个状态的处理方式和`STATE_DONE`类似，当出现这种错误之后，系统将会把这个Task删除了。用户在即将进入`STATE_ERROR`之前也需要像`STATE_DONE`那样，把整个Task的资源删除掉。
* `STATE_BLOCK`，使用这个状态的时候要特别的小心。因为一旦进入了`STATE_BLOCK`之后，这个Task以后再也不会回调了。一般情况下`STATE_BLOCK`与Task的超时一起使用。每一个Task除了进入状态的流程管理，也可以进行超时管理。可以使用Task的`set_timeout_seconds`来进行超时的设置。默认情况下，一个Task是不会超时的，但是如果一量设置了超时，那么在Task超时的时候，这个Task将会被删除掉。父类在遇到超时事件的时候，会调用`OnTimeout()`，可以通过上面的代码看到，在父类当中，默认的`OnTimeout`函数仅仅是使用信息量来通知调用者，Task已经超时。调用者也能够重载这个函数，实现自己在超时的时候的一些处理。除了超时的Task会使用`STATE_BLOCK`状态之外，一些基于网络的Task也可以使用这个状态，这种情况下，Task除了会被自己的Task管理系统所激活，也会被网络事件所激活。而通常为了保持持续接收处理网络件事，常常在Task进入`STATE_START`之后，将Task设置为`STATE_BLOCK`状态。直到Task自身调用自己的`Abort(bool nowake )`或者`Stop`函数，通知Task管理器，将自己删除掉。

Task的状态管理，看起来是比较麻烦，主要是文字描述上面有一些缺陷。

![状态转移图](figure\\TaskStateProcess.png)

如上图所示，这就是整个Task执行过程中的状态转移图。绿色都是状态，而蓝色是其中关于Task里面被执行的过程。可以看到，一般情况下只会执行里面的`ProcessStart`。
* 如果`ProcessStart`返回的是`STATE_RESPONE`，而自己又重载了`ProcesssResponse`的话，那么再会执行自己的`ProcessResponse`。
* 如果进入了`STATE_BLOCK`状态，那么只会等到自己其它地方调用`Abort`和`Stop`的时候才会被删除，或者自己设置了Task超时，等到`OnTimeout`被回调的时候再被删除。
* 如果自己定义了`STATE_NEXT`状态，并且重载了`Process`函数，接下来的流程就由自己控制，但最终还是要回到最基本的这几个状态中来。

其它状态都不会再回调Task自身的函数了。

##VzsipTask的设计
在正常情况下，所有的Task的生命周期都归于TaskRunner来进行管理。那么TaskRunner的生命周期什么时候结束呢？在没有异步事件的情况下。TaskRunner执行完任务之后就会退出，可以认为这个时候TaskRunner的生命周期结束了。但是，如果有的任务出现BLOKED现象，显然一次RunTask是完全不够的。这个时候就需要连续

- Task会进入Block状态，意味着TaskRunner需要不定期的`RunTasks()`，如果没有事件触发的话，`AllChildrenDone`出现，就意味着TaskRunner应该结束了。但是一个Task进入了`STATE_BLOCKED`状态，如果没有外部事件去唤醒这个Task，那么TaskRunner的`AllChildDone`将永远不会Done掉。
- Task进入了BLOCKED状态之后，恢复过来的状态是进入BLOCKED之前的状态。不会把状态回退到STATE_START上去。