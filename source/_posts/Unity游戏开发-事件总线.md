---
title: Unity游戏开发-事件总线
date: 2026-06-02 15:55:15
tags:
  - Unity开发
categories: 编程
cover: https://raw.githubusercontent.com/CC-X-cloud/CC-X-cloud.github.io/refs/heads/main/source/_data/covers/GET.webp
---
# 观察者模式
**官方定义：**  
观察者模式属于行为型设计模式。它的标准定义是：“定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。”
听起来可能有些抽象，接下来我用大白话结合代码场景给大家拆解一下。
## 传统写法痛点：紧耦合地狱
假设我们要写一个天气系统，有一个用来获取数据的 `气象站类`，还有一个负责展示的 `手机类`。  
在没有学习设计模式之前，为了在手机里展示天气，我们通常会直接在手机类中声明并实例化一个气象站对象来获取数据。但问题来了：现实中可能有成千上万部不同的手机！如果每部手机都直接绑定气象站的具体实现，一旦气象站的内部结构发生变化，所有手机的代码都要跟着改。这种高度绑定的代码，维护起来简直是灾难。
## 观察者模式的解法：角色分离与契约精神
观察者模式就是专门解决上述痛点的。核心思想是：将 `手机类` 定义为**观察者**，将 `气象站类` 定义为**被观察者**。我们要实现的效果是——气象站一有数据更新，就主动推送给手机，而手机完全不需要在代码里去声明或强依赖气象站。
具体怎么重构呢？我们可以分三步走：
1. **制定接口契约**：我们用接口为它们定下规矩。规定气象站必须提供“添加观察者”、“移除观察者”和“通知观察者”的方法；同时规定手机只需要提供一个“接收并更新数据”的方法。
2. **建立容器存储**：在气象站内部，我们声明一个列表（集合/表格）来统一存放那些订阅了天气的手机对象。
3. **实现自动广播**：当气象站获取到新数据时，只需遍历这个列表，挨个调用手机们的“更新方法”即可。
通过这样的设计，气象站只认接口不认具体的手机型号，新增或删除设备也无需修改气象站的代码，完美实现了模块间的彻底解耦
# 事件总线
事件总线（Event Bus）系统与观察者模式有着极其密切的血缘关系。简单来说，**事件总线是观察者模式在工程实践中的一种演进、泛化和高级实现形式。**
它们的核心目的都是为了实现组件间的解耦与通信，但在架构层级和解耦彻底程度上存在本质区别：
**1. 核心渊源：同宗同源**  
无论是传统的观察者模式还是事件总线系统，其底层的设计哲学都源自“发布-订阅”的思想。它们都致力于解决同一个痛点：当一个对象状态改变时，如何优雅地通知其他对象做出反应，而不是通过硬编码直接调用对方的方法。
**2. 关键区别：有无“中间人”**  
这是两者最核心的差异所在：
- **传统观察者模式（无中介）**：被观察者（Subject）内部维护了一个观察者列表，当状态变更时，被观察者会**直接遍历并同步调用**观察者的更新方法。虽然引入了接口进行抽象，但被观察者依然知道它正在管理一批观察者，双方存在一定的双向潜在依赖和紧耦合。
- **事件总线系统（强中介）**：它在发布者和订阅者之间强行插入了一个中央枢纽（即事件总线）。发布者只负责把事件丢给总线，订阅者只向总线注册自己关心的事件类型。发布者和订阅者完全互不知晓对方的存在，实现了真正意义上的零耦合（Zero-Coupling）。
**3. 通信机制的升级**
- **观察者模式**通常是基于对象间引用的直接回调，多为同步执行。如果某个观察者的处理逻辑耗时过长，可能会阻塞整个通知流程。
- **事件总线**则将其升级为一种异步的事件路由机制。它不仅支持根据事件类型自动路由匹配，还能提供精细化的线程调度能力（例如决定在主线程还是后台线程处理事件），甚至支持跨进程、跨微服务的分布式通信（如 Spring Cloud Bus 结合 Kafka/RabbitMQ）。
打个通俗的比方：**观察者模式**就像是老师在课堂上直接点名让几个课代表去办事情；而**事件总线系统**则是学校设立了一个专门的“校园广播站”，老师只需要对着麦克风喊话（发事件），谁需要听就自己去收听，彼此完全不需要认识对方。
# Unity开发中的实现
理解了上面的设计逻辑，我们现在看看落实在游戏开发中是怎么实现事件总线系统的。
首先你要明白C#中委托和事件的工作机制的工作机制，事件总线的底层都依赖于 C# 的**多播委托（Multicast Delegate）**。
事件总线运行逻辑
就那玩家移动来说，传统写法中输入模块必须嵌入玩家移动模块，但是有了事件总线我们就可以讲这两个模块分离开来。
事件中心有两部分分别是事件库（存储数据）和方法库（提供订阅，退订等方法），移动控制中调用事件中心获得数据，InputMgr也通过事件中心调用发布和触发方法，这样实现移动控制和输入的强解耦，当获得输入时，输入值可以传给事件库中类，通过构造函数传给类中的变量，然后通过事件中心的触发方法来调用订阅InputMgr类的方法。
答疑：
怎么实现触发方法方法来调用订阅InputMgr类的方法？
这里用委托，可以理解为用了一个指针，通过委托直接指向订阅者的方法，获得输入时，事件总线的方法来指向移动控制，但是这些不重要，**重要的是用这种方法，原本强耦合的对象，可以被拆开，事件中心就相当于一个超级快递员，负责在不同的类中收集和传递数据，让这些类不用见面也可以通信。**
![](GET.webp)
## 代码实现
这里将方法库和事件库分开来，并用上面移动的例子演示运用
### 方法库
{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}
	using System;
	using System.Collections;
	using System.Collections.Generic;
	using UnityEngine;
	
	public class GameEventMgr : SingalBase<GameEventMgr>
	{
	    //核心存储结构：使用字典来管理所有的事件
	    private readonly Dictionary<Type, Delegate> _listeners = new();
		
	    public void Subscribe<T>(Action<T> callback) where T:class
	    {
	        Type type = typeof(T);
	        if(_listeners.ContainsKey(type))
	        {
	            _listeners[type] = Delegate.Combine(_listeners[type], callback);
	        }
	        else
	        {
	            _listeners.Add(type, callback);
	        }
	    }
	    public void UnSubscribe<T>(Action<T> callback)where T:class
	    {
	        Type type = typeof(T);
	        if(_listeners.ContainsKey(type))
	        {
	            _listeners[type] = Delegate.Remove(_listeners[type], callback);
	            if(_listeners[type] == null)
	            {
	                _listeners.Remove(type);
	            }
	        }
	    }   
	    public void Trigger<T>(T eventData) where T:class
	    {
	        Type type = typeof(T);
	        if(_listeners.ContainsKey(type))
	        {
	            var callback = _listeners[type] as Action<T>;
	            callback?.Invoke(eventData);
	        }
	    }
	}
{% endcodeblock %}
### 事件库
{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}
	using System.Collections;
	using System.Collections.Generic;
	using UnityEngine;
	
	public class EventCenter : MonoBehaviour
	{
	
	}
	//用来接受传来的数据
	public class PlayerMoveEvent
	{
	    public float mouseX;
	    public float mouseY;
	    public float horizontalInput;
	    public float verticalInput;
	    public float mouseSensitivity = 5f;
	    public PlayerMoveEvent(float mouseX, float mouseY, float horizontalInput, float verticalInput, float mouseSensitivity)
	    {
	        this.mouseX = mouseX;
	        this.mouseY = mouseY;
	        this.horizontalInput = horizontalInput;
	        this.verticalInput = verticalInput;
	        this.mouseSensitivity = mouseSensitivity;
	    }
	}
{% endcodeblock %}
### 应用示例
#### 订阅者
{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}
public class PlayerMovement : MonoBehaviour
{
    [Header("MovementSpeed")]
    [SerializeField] private float movementSpeed = 3f;
    [Header("RotationSpeed")]
    [SerializeField] private float rotationSpeed = 5f;

    private CharacterController characterController;

	//订阅事件
    private void OnEnable()
    {
        GameEventMgr.Instance.Subscribe<PlayerMoveEvent>(HandlePlayerMove);
    }
    private void OnDisable()
    {
        GameEventMgr.Instance.UnSubscribe<PlayerMoveEvent>(HandlePlayerMove);
    }

    void Start()
    {
        //初始化
        characterController = GetComponent<CharacterController>();
    }

    //角色移动等逻辑主要用CharacterController实现
    public void HandlePlayerMove(PlayerMoveEvent eventData)
    {
        float rotation = eventData.mouseX * rotationSpeed * eventData.mouseSensitivity;
        transform.Rotate(0f, rotation, 0f);

        if (Input.GetKey(KeyCode.LeftShift))
        {
            factor = Mathf.Lerp(factor, 2f, Time.deltaTime * 5f);
        }
        else if(Input.GetKey(KeyCode.LeftControl))
        {
            factor = Mathf.Lerp(factor, 0.5f, Time.deltaTime * 5f); ;
        }
        else
        {
            factor = 1f;
        }
        Vector3 moveVector = (transform.forward * eventData.verticalInput) + (transform.right * eventData.horizontalInput);
        if (moveVector.magnitude > 1f) moveVector.Normalize();
        characterController.Move(moveVector * movementSpeed * factor * Time.deltaTime);
    }
}
{% endcodeblock %}
#### 发布者
{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}
	using System.Collections;
	using System.Collections.Generic;
	using UnityEngine;
	
	public class InputMgr : SingalBase<InputMgr>
	{   
	    private float mouseX;
	    private float mouseY;
	    private float horizontal;
	    private float vertical;
	    public float mouseSensitivity = 5f;
	    private void Update()
	    {
	       mouseX = Input.GetAxis("Mouse X");
	       mouseY = Input.GetAxis("Mouse Y");
	       horizontal = Input.GetAxis("Horizontal");
	       vertical = Input.GetAxis("Vertical");
	    //得到变化并通知订阅者
		GameEventMgr.Instance.Trigger(new PlayerMoveEvent(mouseX, mouseY,
	                                         horizontal, vertical,
	                                         mouseSensitivity));
	    }
	}
{% endcodeblock %}

## PS:
- **委托（Delegate）**：本质上是一种类型安全的函数指针（引用类型），它定义了方法的签名（参数和返回值），允许将方法作为参数传递或动态调用1。
- **事件（Event）**：本质上是委托的封装。它不能脱离委托独立存在，必须依赖委托来指定方法规

## 对比表格
| 对比维度 | 传统观察者模式 (Observer) | 事件总线系统 (Event Bus) |
| :--- | :----------------- | :----------------- |
| 中间层  | 无中介，被观察者直接通知观察者    | 引入事件总线作为消息调度和路由中心  |
| 耦合程度 | 中度耦合（存在接口依赖或引用关系）  | 完全解耦（仅依赖事件总线契约）    |
| 通信方式 | 点对点同步调用为主          | 广播式异步事件路由分发        |
| 适用规模 | 同一进程内、轻量级局部状态联动    | 跨模块、跨线程、跨进程的大型复杂系统 |


# 相关资料
[观察者模式 - ep 2_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1PLJnzaEKB?spm_id_from=333.788.videopod.episodes&vd_source=94e0ab5c7f9607cfa2db45546924ff38&p=2)
[微软官方# 探索 C# 中委托和事件的关系](https://learn.microsoft.com/zh-cn/training/modules/create-manage-events/5-explore-delegates-events-relationship-csharp)
