# Davinci Adaptive IDE 的使用
* 开发工具安装
#SOME/IP概述
* 服务接口(Service Interface)
  * Method - 方法表示提供者根据一个或多个请求执行的功能 
    - 客户端向服务端发送请求报文
  * Property/Field - 属性（字段，属性）表示由提供者托管的一段数据，它向一个或多个消费者公开获取或设置方法。 消费者可以选择性的接收字段值更改的通知。
    - Setter/Getter 客户端请求获取/设置某一属性/状态
    - Notifier 客户端订阅某一属性/状态后，Server端发布该服务。发布条件同Event,不同是订阅后Server端会立即发送此Field的内容
  * Event - 事件表示对一条数据的更新。提供者决定何时发送此更新并将其事件传达给一个或多个消费者。 
    - 客户端订阅一个服务，服务端发布该服务
    - 状态: on change/Cycle
    - 状态直: on change/Cycle/事件值变化超过预期设置范围

#Skeleton Class
原文链接：https://blog.csdn.net/kuno_y/article/details/122430912
* 代码生成和构造
* 提供服务实例
* 论询和事件驱动模式
  * Polling Mode
  * Event-Driven Mode
* Methods
* Events
* Fields
  * 注册Setter和Gettter
#代码生成和构造
skeleton class 直接从ARXML生成，开发人员要继承这个虚类，然后在子类里实现架构设计的本服务提供的方法。
skeleton实例的构造函数重载可以从
* ara::com::InstanceIdentifier、实例标识符 
* ara::com::InstanceIdentifierContainer
  （multi-binding）实例标识符容器
* ara::core::InstanceSpecifier 实例说明符
* void OfferService(); 提供服务
* void StopOfferService(); 停止提供服务
都是幂等（idempotent）函数。随便调用多少次都和调用一次效果相同。

从同一个skeleton class可以部署很多的服务实例。当然了每次构造都要基于不同的identifier。在构造前会有一个静态类成员方法来检测该identifier是不是独一无二的。如果基于同一个identifier构建新实例，之前已经构造好的需要先销毁。这样也可以理解为什么不让使用复制构造函数和复制分配（copy construction nor copy assignment）。同样的identifier会让method调用之类的变得不确定。
下面来看三种不同的构造输入： 

  * ara::com::InstanceIdentifier

​    依据具体的binding描述符创建服务

  * ara::com::InstanceIdentifierContainer

​     multi-binding…依据容器包含的多个描述符创建实例。多绑定

  * ara::core::InstanceSpecifier

    依据 在service manifest中查找ara::core::InstanceSpecifier获得的 instance identifier(s)来创建服务实例，如果在manifest中这个InstanceSpecifier是map到了多个binding的描述符，那么就是multi-bingding.
在构造实例后，该实例会对用户不可见，以防止潜在的用户调用它的method。直到在服务的程序中调用

OfferService API。
Offering Service instance

在初始化服务提供实例，并确保state之类的都到位之后，调用OfferService()来提供服务。
调用OfferService()，哪怕还没返回，service就已经开始接收服务请求了。
如果不想提供服务了，调用StopOfferService()。该API返回之后不再接收服务请求。
AP要求供应商在服务实例的析构函数中嵌入StopOfferService()。

#轮询和事件驱动模式
服务调用何时发生是不可确定的。

对纯粹的事件驱动来讲，CM要生成events calls。然后把这些events转换成具体的method调用，再调用具体的service实现的service methods。

这样的结果是：
    直接调用服务Method是最快的，延时只受到整体的负载和IPC机制的限制。
    包含服务实例的OS进程上下文切换可能会很频繁且不确定。降低总的产出。

我们还支持纯纯的轮询方式。服务提供者可以显示调用ara::com的API来执行单个 call event。
构造函数的第二个参数是描述生成实例的处理方式（轮询还是事件之类的）

```c++
enum class MethodCallProcessingMode { kPoll, kEvent, kEventSingleThread };
```

在服务实例的整个生命周期内，该服务实例都是同一种mode。
#Polling Mode

设为kPoll，CM不会异步调用提供的服务method。
在服务实例的代码中显式调用：

可以从CM的待处理Method call队列中取一条来执行。本API仅限Poll模式调用。

用Future的好处是在执行完之后可以得到提醒。可以使用future里的then(), 来判断在条件成熟的情况下，再调用ProcessNextMethodCall()，实现链式调用。

在有外部调用的时候，ara::core::Future的返回值会被CM设置为true。方便开发人员在队列里有调用请求排队的时候调用ProcessNextMethodCall。如果上一个调用已经返回false了，再调用可能不会做任何事情。（也有可能在返回的这一点点时间里来了一个调用，可能性很小）。
#Event-Driven Mode

设为kEvent或者kEventSingleThread。CM就会在收到服务消费者请求的时候异步的调用服务实例的Method。
与轮询模式不同，服务用户会在调用method的时候触发服务提供者的活性（可以唤醒？是这个意思吧）。
KEvent让CM可以并发调用服务Method。比如有来自不同服务用户的一个MethodA和两个MethodB同时到达，CM会从线程池里拉3个线程来并发处理这些Method调用请求。
kEventSingleThread就是CM一次调用一个service method。CM需要维护一个队列来缓冲请求，然后一个个调用method处理。
设计两种方式是为了效率和节省资源考虑。如果Method调用间有大量的数据同步需求(比如先后依赖)，建议使用kEventSingleThread避免大量空闲线程等待关键线程数据结果。
Methods

skeleton 生成类中。Method定义为纯虚函数。必须在子类中复写。
输入参数就直接映射到了Method的输入参数。返回值使用了ara::core::Future。
为什么要用Future呢，是因为我们会使用线程池来处理做Method的工作，调用的Method入口函数需要与线程池真正干活的线程同步（就是要等他干完活才能返回）。

面对多线程编程，和传统嵌入式的编程方式真的是有些不一样啊

下面来看两种实现方式 “同步”和“异步”：
```c++
using namespace ara::com;
/**
* Our implementation of RadarService
*/  
class RadarServiceImpl : public RadarServiceSkeleton 
{
public:
    Future<AdjustOutput> Adjust(const Position& position)
    {
        ara::core::Promise<AdjustOutput> promise;
        // calling synchronous internal adjust function, which delivers results
        struct AdjustOutput out = doAdjustInternal(
                        position,
                        &out.effective_position );
        promise.set_value(out);
        // we return a future from an already set promise...
        return promise.get_future();
    }
 private:
    AdjustOutput doAdjustInternal(const Position& position) {
    // ... implementation
    }
 }
 ```
 Method是一个入口，又调用了一个私有成员函数来执行操作（解耦？）。在调用doAdjustInternal得到返回值的时候，Method的输出就已经可以确定了。然后使用promise的set_value来获取out,再通过return来返回promise.get_future()，基于此让返回值构建函数返回的future。
调用这个Method的caller,可以马上通过Future::grt()获得结果。这个结果是直接通过调用方法函数得到的，不会存在阻塞。

下面是另外一种异步worker thread的方式。
```c++
 using namespace ara::com;

 /** 
 * Our implementation of the RadarService
 */ 
 class RadarServiceImpl : public RadarServiceSkeleton 
 {
 public: 
    Future<AdjustOutput> Adjust(const Position& position)
    {
        ara::core::Promise<AdjustOutput> promise;
        auto future = promise.get_future();

        // asynchronous call to internal adjust function in a new Thread
        std::thread t(
        [this] (const Position& pos, ara::core::Promise prom) {
        prom.set_value(doAdjustInternal(pos));
        },
        std::cref(position), std::move(promise)).detach();

        // we return a future, which might be set or not at this point...
        return future;
    }

 private:
    AdjustOutput doAdjustInternal(const Position& position) {
    // ... implementation
    }
 }
```

