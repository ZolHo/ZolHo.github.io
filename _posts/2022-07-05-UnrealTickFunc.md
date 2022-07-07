---
layout: post
title: "UE中的Tick实现机制浅析"
subtitle: ''
author: "ZolHo"
header-style: text
tags:
  - UE
---

## 一、Actor如何实现Tick

当我想探寻下Actor是如何实现Tick时，我想到了两种方法。一是给Tick函数打个断点，二是我们知道Actor可以通过修改`PrimaryActorTick.bCanEverTick`实现开关Tick功能，我们也可以从这里入手寻找相关的代码。

我们先看看控制Tick开关的成员变量`PrimaryActorTick`，它是一个`FActorTickFunction`类型的结构体，继承自`FTickFunction`，并且相比父类只多了个指向Actor的指针`Target`，可见大部分东西都在父类`FTickFunction`中。大致浏览下该结构体的内容，我把需要的部分画出：![Actor and FTickFunction](/img/note/2022-07-06-11-58-30.png)
可以看到，`FTickFunction`有着`注册Tick函数`和`执行Tick`两个看上去很重要的功能。

先来看`ExecuteTick`，它是一个虚函数，交给子类实现。比如`FActorTickFunction`的实现中，它会调用`AActor::TickActor`，再调用Actor的Tick函数。假如如注释所言，它是Actrually execute the tick的，那么说明真正调用Tick函数的其实是Actor中的成员变量`PrimaryActorTick`。

想要知道`FTickFunction`是如何帮助Actor进行Tick的，我们需要继续看看它另一个成员函数`RegisterTickFunction`。这个函数很短，主要就是调用了`FTickTaskManager::Get().AddTickFunction(Level, this);`，而这个函数则根据传入的Level，拿到Level的一个`FTickTaskLevel`类型的成员变量，并调用该变量的`AddTickFunction`函数，把`FTickFunction`保存到`FTickTaskLevel`的一个保存TickFunction的集合`AllEnabledTickFunctions`中。

简单来说，注册过程就是把Actor的变量`FActorTickFunction`保存到Level中的一个变量里的集合(TSet)中。对注册函数打断点可观察到，Actor的Tick函数是在BeginPlay阶段注册，而组件会在Actor生产成时调用`RegistAllComponent`时注册。可想而知，这个`FTickTaskLevel`以关卡为单位保存着`FTickFunction`的集合，当世界的时间开始流动，只要遍历Level就可以拿到所有要执行的Tick函数。并且，这样的设计实现了**Tick系统**和**需要Tick的对象**之间的解耦，Tick系统不需要知道对象是什么类型，只要它持有一个`FTickFunction`对象进行注册，就可以被Tick系统进行管理。

不过事情并不是一个for循环就能搞定的，现在我们还不太清楚`FTickTaskManager`和`FTickTaskLevel`到底是什么样的，不过我打算先用断点的方式看一看Tick时候的函数的调用栈，来看一下其他我们没涉及到的部分：
![Tick函数调用栈](/img/note/2022-07-06-16-01-46.png)

如图是一个PlayerController调用Tick函数时候的调用栈，先来大概的看看我分的4个部分。第一部分是引擎运行后执行引擎的Tick循环，而我们关注的Actor的Tick都是在`UWorld::Tick`中实现的。第二部分是`FTickTaskManager`这个管理者会把当前指定**分组**的Tick任务交给**多线程任务系统**`TaskGraph`去并行的执行任务，也就是第三部分。第四部分可以看到是`FTickFuctionTask`去调用了上文所说的`FActorTickFunction`执行Tick，而这个`XXTask`则是任务系统所需要的对一个任务的包装，目前和Task有关的方法我们应该可以无视。

可以看到，第4部分和我们之前所想是一致的，23是任务系统相关，所以我们只要再研究下`UWorld::Tick`的流程就可以大致理解Actor的Tick过程了。

```cpp
void UWorld::Tick( ELevelTick TickType, float DeltaSeconds ) {
    
    for (int i = 0; i < LevelCollections.Num(); ++i) {

        FTickTaskManagerInterface::Get().StartFrame(this, DeltaSeconds, TickType, LevelsToTick);

        RunTickGroup(TG_PrePhysics);
        RunTickGroup(TG_StartPhysics);
        RunTickGroup(TG_DuringPhysics, false);
        RunTickGroup(TG_EndPhysics);
        RunTickGroup(TG_PostPhysics);

        // Tick Timer 和 TickableObject
        GetTimerManager().Tick(DeltaSeconds);
        FTickableGameObject::TickObjects(this, TickType, bIsPaused, DeltaSeconds);

        RunTickGroup(TG_PostUpdateWork);
        RunTickGroup(TG_LastDemotable);

        FTickTaskManagerInterface::Get().EndFrame();
    }
}
```

上面是`UWorld::Tick`部分代码，可见在执行Tick前后会进行StartFrame和EndFrame，中间则按分组依次执行。关于分组TickGroup，其实是为一次Tick划分了若干阶段，阶段间串行执行，同一阶段的TickTask则可以并行执行。可以查看[官方文档](https://docs.unrealengine.com/4.26/zh-CN/ProgrammingAndScripting/ProgrammingWithCPP/UnrealArchitecture/Actors/Ticking/)了解每个阶段的具体解释。

在`StartFrame`流程中可以看到`FTickTaskManager`会将所有Level中的`FTickTaskLevel`取出来自己保存，这里面就有这全部Actor（Component同样）的`FTickFuction`。如此一来，整个大概的流程算是串联起来了，虽然很多问题比如`FTickTaskSwquencer`的作用还没有搞清楚...不过先到此为止。下面简单总结下：

**总结**： Actor会把Tick相关的信息保存在自己`FActorTickFunction`类型的成员变量中，当BeginPlay时把它向Level注册。而World会在它的Tick函数中通过Level获取这些注册过的TickFunction，然后按分组顺序让`FTickTaskManager`交给`TaskGraph`多线程地执行它们。

## 二、Component和TickAbleObject

再来看看同样可以进行Tick的组件以及用户自己实现的TickAbleObject。

和Actor类似的，`ActorComponent`中有着`FActorComponentTickFunction`类型的成员变量`PrimaryComponentTick`。它也继承自`FTickFunction`，并且只多了个指向Component的指针。所以它的Tick实现与Actor如出一辙，只是注册时机有所不同。

在UE4中，Gameplay对象提供了虚函数Tick供我们重载，然后如上文所说进行每帧执行。若我们想让Object或者原生C++类进行Tick可以通过继承`FTickableGameObject`类，然后实现它的纯虚函数`Tick`和`GetStatId`即可自动每帧执行Tick函数。

很神奇，甚至不用手动实例化对象就会执行Tick，这是咋做到的呢？我们可以看到上面World Tick的代码中，它调用了`FTickableGameObject::TickObjects`，它是一个静态成员函数。可以想象，UE应该是用了某种手段把所有的子类实例统一管理，我们可以尝试从该函数以及构造函数寻找线索。

```cpp
FTickableGameObject::FTickableGameObject()
{
  FTickableStatics& Statics = FTickableStatics::Get();
  Statics.QueueTickableObjectForAdd(this);
}
```

上面的代码就是`FTickableGameObject`的构造函数（简化），它会叫全局单例`FTickableStatics`过来，并把自己保存到它的一个TSet中。而`TickObjects`函数则同样把这个单例叫过来，然后取出它保存的TickableObject，根据它的设置走不同的分支，最终调用了这些对象的Tick函数。

并且由于UE会为类自动生成CDO（类默认对象），所以我们不用手动实例化就可以有一个默认的对象进行Tick。这里可以做个实验，如果我们手动实例化，它应该会多执行相应的对象的Tick。所以TickableObject的机制看上去也不复杂（也可能是我的问题，这段没参考其他文章），就是在构造时保存到全局对象中，然后在Tick的过程中取出调用。

> 参考文章：  
> [Unreal TickFunc调度 - 知乎](https://zhuanlan.zhihu.com/p/467438700)  
> [硬核分析：Unreal Tick 实现 - KM](https://www.163.com/dy/article/H4EVI8360518R7MO.html)  
> [UE4 Tick机制解析 - KM (无外链抱歉)](https://km.woa.com/group/20930/articles/show/429077)