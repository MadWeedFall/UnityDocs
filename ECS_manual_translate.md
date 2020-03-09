# 概论
实体组件系统 Entity Component System (ECS) 是 Unity 面向数据技术栈的核心。顾名思义，ECS 包括三个组成部分

+ Entities(实体) 是大量充斥在你的游戏或软件的实体，或说是物体
+ Components(组件) 是与你的各种实体相关联的数据，这些数据由数据本身组织在一起，而不是由实体来组织（这种组织方式的区别是面向数据的设计与面向对象设计最根本的区别之一）
+ Systems(系统) 是将组件数据从当前状态转变为下一个状态的逻辑——例如，一个系统可以用来将当前所有正在移动的实体的位置更新为实体的速度与自上一帧开始的时间差的乘积

# ECS核心内容

## ECS相关概念

实体组件系统 Entity Component System (ECS) 架构将定义（Entities），数据（Components），行为（Systems）独立区分开。架构聚焦于数据。 系统通过读取由实体索引的组件数据流，将数据的状态从某种输入状态转变为某种输出状态。

下图展示了这三个基本部分是如何工作的
![ECS三部分如何工作](https://raw.githubusercontent.com/MadWeedFall/UnityDocs/master/img/ECS_manual_translate/ECSBlockDiagram.png)

在上图中系统读取位移和旋转组件，将它们相乘用于更新与它们关联的本地坐标组件。
实体A与实体B都有渲染器组件，而实体C没有，这并不影响系统，因为系统并不关注渲染器组件（你可以创建一个需要渲染器组件的系统，在这种情况下，系统会忽略实体C；或者你也可以创建一个排除带渲染器组件实体的系统，这样系统就会忽略实体A和实体B）

#### 原型 Archetypes
某种特定的组件类型的组合称作原型。例如，某种3D对象可以有一个代表世界位移的组件，一个代表线性移动的组件，一个代表旋转的组件，还有一个用于视觉展现的组件。这种3D对象的每一个实例都与一个单独实体相关联，但由于他们都有一系列相同的组件，这些对象实例都可以归类为同一个原型。

如上图所示，实体A和B都属于原型M，而实体C属于原型N。

可以通过添加或者删除组件来任意改变实体所属的原型。例如，如果删除B的渲染器组件，B就会归类到原型N中。

#### 内存块

实体的原型决定了实体的组件存储在哪里。ArchetypeChunk对象表示一整块内存，这块内存中存储的都是相同原型的实体。如果一块内存满了，新的内存块是按照相同原型的新建实体来分配的。 如果通过添加或删除组件来改变一个实体的原型，那么这个实体会被转移到别的内存块中。

这种组织方式在原型和内存块之间建立了一种一对多的关系。这同时意味着只需要查找现有的原型就可以找到所有持有特定组件组合的实体，而原型的数量相对所有实体来说少的多，通常原型数量很小，实体的数量会很大。

注意实体的组件不是按照某种特定顺序存储的。向某个原型添加实体时，实体会被放到该原型的第一个能存得下的内存块中。内存块是紧密排列的，当某个实体从原型中被删除掉，内存块中排在最后的实体中的组件会被移动到组件数组中新增的空槽中。

#### 实体查询

可以使用EntityQuery来确定系统需要处理哪些实体。一次实体查询会在现有的原型中查找符合条件的结果。可以设置以下查询条件
+ All -- 原型必须包含在All类别(category)中的所有组件类型
+ Any -- 原型必须至少包含一个在Any类别中的组件类型
+ None -- 原型必须不包含任何在None类别中的组件类型

一次实体查询会产出包含本次查询指定类型组件的内存块列表。可以使用IJobChunk对这些内存块中的组件直接进行遍历，IJobChunk是一种特殊的ECS事务。也可以使用包含隐式查询IJobForEach或 non-job for-each loop来遍历。

## 实体

实体是ECS架构的三个关键组成元素之一。实体代表了游戏或软件中各种各样的“物体”。实体既没有行为，也没有数据，它只是定义了哪些数据可以需要放到一起。系统提供行为，而组件则用于存储数据。

实体本质上就是ID。你可以认为实体是一种默认连名字都没有的超轻量级的GameObject。实体ID是固定的。实际上这些实体ID是存储组件引用或是其他实体引用的唯一固定的方式。

EntityManager管理在World（世界）中的所有实体。EntityManager维护一个实体列表，同时组织实体相关的数据用于优化性能表现。

虽然实体本身没有类型，但是多组实体可以通过关联的数据组件的类型来归类。当你创建实体并添加组件到实体时，EntityManager记录已有实体中的各种唯一的组件组合方式。这种唯一的组合方式被称为ArcheType（原型）。当添加组件到实体的时候，EntityManager会创建对应的EntityArchetype结构体。可以通过已有的各种EntityArchetype创建属于对应原型大的新实体。也可以预先创建一个EntityArchetype然后用它来创建实体。

#### 创建实体

通过Unity编辑器创建实体是最简单的创建实体的方法。可以在运行时将场景中的GameObject和Prefab转化成实体。当需要在游戏或软件中动态创建实体时，也可以通过创建生成系统通过一个job（事物）生成多个实体。当然，也可以通过使用EntityManager.CreateEntity方法来一次创建一个实体。

#### 使用EntityManager创建实体

使用EntityManager.CreateEntity方法来创建实体，所创建的实体会在和EntityManager所属的同一个World（世界）对象中被创建。

你可以通过下列方式一个一个的创建实体：

+ 使用CommonType对象数组创建带有组件的实体
+ 使用EntityArcheType创建带有组件的实体
+ 使用Instantiate，拷贝一个已存在的实体，包括该实体当前的数据
+ 创建一个不带组件的实体，然后在向其添加组件。（可以在实体创建后立即添加组件或在需要的时候添加额外的组件。）

你也可以通过下列方式一次创建多个实体：
    
+ 使用CreateEntity在一个NativeArray中填充属于相同原型的实体。
+ 使用Instantiate在一个NativeArray中填充已存在实体的拷贝，包括实体中当前的数据。
+ 使用CreateChunk显式创建多个包含指定数量特定原型的实体的内存块

#### 添加删除组件

当时实体创建完成后，如果添加或删除组件，受影响实体的原型和EntityManager必定会讲变更数据转移到一个新的内存块中，同时压缩原来内存块中的组件数组。

对实体的修改会造成结构上的改变——当添加或删除组件时改变了SharedComponentData的值，从而销毁了实体。这一操作不能在Job（事务）中进行，因为会造成事务处理的数据不可用。相反，你应当向EntityCommandBuffer中添加命令来实现这些修改，并且在事务执行完成后这个命令缓冲区才会执行。

EntityManager 对单个实体和NativeArray中的多个实体都提供了删除组件的方法。详见Components（组件）一章。

#### 遍历实体

遍历拥有相同一系列组件的实体，是ECS架构设计的核心内容，参见访问实体数据一章

#### World（世界）

一个World包含一个EntityManager和一系列ComponentSystems。你可以随意创建多个World对象。通常你会创建一个逻辑仿真World和一个用于渲染或展示的World

进入Play Mode的时候默认会创建一个World，工程中的各种可用的ComponentSystem对象都会填充到这个World中。不过你也可以取消默认World的创建，并通过全局宏定义将其替换成你自己的代码。

\#UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP_RUNTIME_WORLD 取消默认运行时世界的创建

\#UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP_EDITOR_WORLD 取消默认编辑器世界的创建

\#UNIT_DISABEL_AUTOMATIC_SYSTEM_BOOTSTRAP 同时取消上面两种默认世界的创建

+ 默认世界创建源码（参见代码文件：Packages/com.unity.entities/Unity.Entities.Hybrid/Injection/DefaultWorldInitialization.cs）
+ 自动启动入口点 （参见代码文件：Packages/com.unity.entities/Unity.Entities.Hybrid/Injection/AutomaticWorldBootstrap.cs)

## 组件
组件是ECS架构的三个关键组成元素之一。组件代表游戏或软件中的数据。实体本质上是用于编号组件集合的标记。系统提供行为。

具体来讲，ECS架构中的组件是有以下各种“特征接口”之一的结构

+ IComponentData —— 用于泛用组件（general purpose）和内存块组件（chunk components）
+ IBufferElementData —— 用于在实体中关联动态缓存（dynamic buffers）
+ ISharedComponentData —— 用于根据原型中的值分类或编组实体。详见共享组件数据（Shared Componet Data）
+ ISystemStateComponentData —— 用于在实体中关联系统相关的状态，并用于检测单个实体的创建和销毁。详见系统状态组件（System State Components）
+ ISharedSystemStateComponentData -—— 是共享数据和系统数据的结合。详见系统状态组件（System State Components）

EntityMananger将实体中独特的组件组合组织成原型。它将所有相同原型的实体的组件存储到被称为内存块的内存区域中。在指定内存块中的实体都具有相同的组件原型。

![ArcheTypeChunkDiagram](https://raw.githubusercontent.com/MadWeedFall/UnityDocs/master/img/ECS_manual_translate/ArchetypeChunkDiagram.png)

上图展示了组件数据如何根据原型存储在内存块中。共享组件和内存块组件不是这么处理的，因为它们是在内存块之外存储的，这些类型的组件的单一实例在所有可应用内存块中应用到所有的实体上。你也可以选择将动态缓存存储在内存块之外，通常在查询实体的时候可以像处理其他组件类型一样处理动态缓存。

## 泛用组件

Unity中的组件数据ComponentData（在标准的ECS定义中称为组件）是仅包含实体数据实例的结构。ComponentData不能有除访问结构内数据的工具方法外的其他方法。所有的游戏逻辑和行为都应当在系统中实现。如果按照老的Unity系统定义，这和老的Component类有点相似，不过ComponentData只能包含变量。

Unity ECS提供了一个叫做IComponent的接口用于实现你自己的代码。

#### ICompoentData

旧版的Unity组件（包括MonoBehaviour）是面向对象模式的类，其中包含数据和方法用于定义行为。IComponentData则是纯粹的ECS风格组件，这意味着组件不定义行为，组件仅仅是数据本身。IComponentData与其说是一个类倒不是如说是一个结构体，也就是说它默认是值拷贝而不是引用拷贝。你会经常用到如下代码模式来修改数据：

```c#

var transform = group.transform[index]; // Read

transform.heading = playerInput.move; // Modify
transform.position += deltaTime * playerInput.move * settings.playerMoveSpeed;

group.transform[index] = transform; // Write

```

IComponentData 结构不能包含托管对象的引用。因为所有的ComponentData的生命周期都在无垃圾回收检测的内存块内存中。

详见源码文件：/Packages/com.unity.entities/Unity.Entities/IComponentData.cs