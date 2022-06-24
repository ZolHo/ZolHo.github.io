---
layout: post
title: "Actor管理与生命周期学习与理解"
subtitle: ''
author: "ZolHo"
header-style: text
tags:
  - UE
---

> 注：以下内容均为自己学习过程中结合实验、源码和他人文章进行的理解和推测，一定有不准确和疏漏，感谢指正。参考源码版本4.27

## 一、World，Level，PersistentLevel和Actor的关系

总所周知，Actor作为UE游戏世界基本的独立单位，我们能在游戏内看到的所有东西都是继承自AActor类。那么它们是如何存在于这个世界的呢，我们先看下World、Level和Actor三者的概念和关系。

World表示的是一个容纳着若干Actor的游戏世界/地图，我们本可以将Actor都保存在其中，但是对于一般的Actor却并没有在World中保存引用。这是因为考虑到我们将一个世界中的内容拆分为若干集合，那么一来我们可以方便实现运行时动态加载、卸载部分地图，方便制作DLC等不会一开始就展现给玩家的内容。二是开发者可以并行进行不同部分的开发而不用担心同时修改同一个文件产生冲突。*(推测)*

所以UE添加了中间层Level，让Level作为Actor的容器，World作为Level的容器。我们对Level的修改最终会通过序列化World保存在`.umap`文件当中，并在加载World或Level的时候反序列化出来。
再来看看源码中，World类中有着成员变量`PersistentLevel`指向World中的主关卡，或叫做持久化关卡，这是什么东西呢？

- 当我们在编辑器新建一个Level的时候，GameInstance会卸载当前的World，创建个新的World并将新Level作为主关卡。*（比如UnrealEngine::LoadMap, 12817行）*
- 当我们保存它的时候，会新建一个`umap`后缀的文件，将World序列化。这个文件就保存了主关卡和它的所有信息。*（UWorld::Serialize, 406行）*
- 当我们设置其他Level作为其**子关卡**的时候，序列化并不会保存子关卡的Actor，而是保存一个引用信息。*(推测)*

由此可见，主关卡和map文件是一一对应的，而map文件又是一个World的序列化（特指Editor类型的World，运行时的World不会有序列化，大概），也是一一对应。虽然说World是Level的容器，但World和它的主关卡几乎算是同等级的存在。每一个Level在它自己的World中就是主关卡，在运行时加载其他Level时候，则把新加入的Level保存在`Levels`数组中，*(UWorld::AddToWorld, 2448行)* 于是World可以通过此数组间接访问到世界中的所有Actor。

总结一下，我们在编辑器中创造的关卡会和当前的世界一起保存在`.umap`类型的`Package`中，当在这个世界添加子关卡时，会调用`UWorld::FindWorldInPackage`从对应关卡的`Package`中取得它的World中的主关卡资源载入，这便是UE中World和Level的关系。我自己简单设想了一下将World和Level完全分离的设计，将`WorldSettings`从Level转移至World中，让World控制一些Level、Actor无关的配置，感觉也可做为一种设计方案。（当然是作为初学者的瞎想）

## 二、Actor的创建

在上文主要讨论World和Level的关系之后，我们来再把重心放到Actor的生命周期上来。参考官方的[Actor的生命周期图](https://docs.unrealengine.com/5.0/Images/programming-and-scripting/interactive-framework/unreal-engine-actors/unreal-engine-actor-lifecycle/ActorLifeCycle1.png)，可以看到Actor的主要生成方式有两类，一类是从已有的资源进行复制或者加载，另一类是真的创造新的Actor。我们依次看一看这几种方式。不过在继续下文之前，我们首先最好知道World存在多种类型，可以在`EngineTypes.h`中的`EWorldType`查看。例如我们在编辑器中编辑的是`Editor`类型的World，点击Play之后运行的则是`PIE`类型。

### Play in Editor

图中告诉我们，当我们在`Play in Editor`的时候，所有的Actor会复制到新World中。此方式存在前提是，我们新建`PIE`世界时，游戏是运行在编辑器进程中，而非真正的创建游戏进程。**个人认为**存在此加载方式的原因有两点，一是编辑器中的World可能还在内存中存储，并未保存在磁盘上，这种方式可以省去写回磁盘的步骤；二是既然内存中已经加载了我们需要运行的Actor，直接从内存复制显然比从外存读入快的多。（其实就是能从内存找到，就不去外存）此方式会调用`UWorld::GetDuplicatedWorldForPIE`，名字很直白，但我目前没仔细看`TODO`。在后面要提到的`LoadMap`的过程中，也会在磁盘加载map前判断是否可以复制，然后从`WorldContext`中找到对应的World进行复制。*(UEngine::LoadMap, 12965行)*

### LoadMap 和 AddToWorld

这两个函数名字很直白，分别是加载map和把level添加到World。它们分别位于`UEngine`和`UWorld`，这也映照了map和world是对应的，所以加载map需要更高层次的Engine来做，而添加子关卡交给World就足够了。据官网图片所言，它们会从磁盘上加载Actor，但事实上它们也是会先查看能否在内存中找到相应的package加载，没有捷径最终才会访问磁盘 *(UEngine::LoadMap, 13029行)*。虽然看上去功能很直白，但我实际调试起来还是有很多问题：

一是我发现这两个功能并不是靠直接通过函数调用来实现的，而是在其它地方产生需求，之后在Tick的过程中发现有需求，再调用`LoadMap`和`AddToWorld`完成。例如`LoadMap`会被`UEngine::TickWorldTravel`调用，它会根据`WorldContext`的信息来决定是否要做出加载map的动作。这让我不知道在哪去找修改`WorldContext`的地方，好在搜索资料的过程中发现[《InsideUE4》](https://zhuanlan.zhihu.com/p/23167068)中其实有提到是`UGameplayStatics::OpenLevel`中调用的`UEngine::SetClientTravel`进行的。而`AddToWorld`则似乎是通过标记流关卡状态，来进行加载或卸载的`(UWorld::UpdateLevelStreaming)`，并且它还可以将工作分布在多帧中完成，到底是不是这样的还需要再学习调查 `TODO`。

二是似乎地图的加载所走的途径和是否联机，是否流关卡，World的模式等都有关联。这让我调试的时候很混乱，并且我也不知道在**非PIE**的World下到底是如何加载地图的，所以这一块只能在未来的学习中慢慢深入了`TODO`。

上面说了那么多，我们再把注意力放到产生Actor上。以我现在的水平，很难说清楚Actor经复制、加载后的准确状态，不过可以俯瞰下，Actor现在应该是和编辑器中类似，已经在内存中拥有自己的地盘，并且是执行完成构造函数的状态（这点在后文编辑器中添加actor也可以印证）。那么它想要在运行时的游戏世界中运行，大概还需要完成将自己的组件向World注册，让World管理Actor及其组件的运行状态，调用一些生命周期相关的事件(PreXX, PostXX)，随世界BeginPlay并一起Tick。上述内容会如官图所说，在`UWorld::InitializeActorsForPlay`中调用`ULevel::RouteActorInitialize`完成。

### SpawnActor 和 SpawnActorDeferred

最后，我们再来看看真正的万A之源`UWorld::SpawnActor`，无论是往编辑器中拖入一个Actor，还是运行时生成，以及调用`SpawnActorDeferred`延迟生成Actor，最终都是调用的`SpawnActor`。在它内部，通过传入的`SpawnParameters`以及一些外部条件，决定了生成Actor的时候走向不同的分支。我们先来看看当我们在编辑器中拖入一个Actor会发生什么。

首先，对于纯C++和蓝图两种Actor拖入时，会走不同的创建路径，分别是`UEditorEngine::AddActor`和`UActorFactory::CreateActor`，但最终会指向`SpawnActor`。不过看上去在生成前并没有对`Parameters`做特殊的修改，所以生成流程和运行时不同之处在于没有调用`BeginPlay`（这是我目前唯一确定的不同）。

> 这部分越学越乱，需要再去学习，之后再完善....To Be Continue...
