---
title: Unity游戏开发—基础框架概述
date: 2026-06-08 15:57:34
tags:
  - Unity开发
cover: https://raw.githubusercontent.com/CC-X-cloud/CC-X-cloud.github.io/refs/heads/main/source/_data/covers/UNJKGS.webp
categories: 编程
---
# 游戏开发基础框架是什么？
比较官方的叙述是：从底层设计思想来看，Unity 框架的本质是“**经验的抽象沉淀**”。它是一套基于 Unity 引擎特性设计的“**代码规范与功能模块集合**”。与直接编写业务代码不同，框架更侧重于“如何优雅地组织代码”，通过预设的结构和接口，解决项目开发中的共性问题（如模块耦合、功能复用、逻辑混乱等），从而让开发者将精力聚焦于核心玩法的实现。
简单来说游戏框架是解决游戏开发中如何组织代码（不让代码逻辑变成史山）并且每个项目几乎都存在的问题（每个项目都要写一遍的），其底层原理是设计模式和软件架构思想等知识。
在设计的时候其主要要考虑的是，游戏类型和需求，可维护和拓展性，通用性和性能等等

# 游戏开发基础框架的构成
Unity框架在宏观层面分为4个层级：
基础设施层：完全和游戏玩法设计无关，通常包含最基础的代码规范和通用能力，比如单例模式基类，对象池，公共Mono等。
核心服务层：为上层业务提供标准化的底层服务接口，不关心具体的游戏玩法，但是所有的业务模块都会用到，比如事件总线，资源管理系统等。
业务逻辑层：开发者的主要阵地，实现游戏逻辑和具体玩法，代码高度定制化，且频繁调用第二层提供的核心服务，比如战斗系统，角色控制系统，UI系统等。
框架控制和辅助层：负责统筹全局，提升团队开发效率，包含有，全局控制：框架入口（初始化所有模块和调度），宏观状态机：游戏流程状态机，开发提效：自动化工具。
为了对接策划和游戏中的现实需求我们还需要对业务层面分析。
对业务逻辑层级进行横向的解刨按照功能层级划分为：
UI框架层，核心逻辑框架层，资源管理层，数据管理层，网络层

PS：框架控制和辅助层的核心脚本是GameManager（不一定叫这个）主要用途是提供统一的调用接口，可以避免管理器初始化，循环依赖等问题，避免出现调用某个管理器时为空的情况，和管理器之间直接调用产生的问题。
代码示例
{% codeblock lang:csharp %} 
// 没有之前 
	// 玩家受击时，想扣血并播放音效 /
	/ 痛苦地到处找对象，性能极差且容易报错
	AudioSource audio = FindObjectOfType<AudioSource>(); 
	audio.Play(); 
	UIManager ui = GameObject.Find("UIManager").GetComponent<UIManager>(); ui.UpdateHp(hp); 
	// 有之后 
	// 玩家受击时，直接找 GameManager 
	GameManager.Audio.Play("HitSound"); 
	GameManager.UI.UpdateHp(hp); 
{% endcodeblock %}

# 引用和参考
[GameFramework解析：开篇 - 知乎](https://zhuanlan.zhihu.com/p/426136370)
[【唐老狮】Unity程序基础框架（重置版）—1.概述](https://www.bilibili.com/video/BV1Nu4y1M7Rt/?share_source=copy_web&vd_source=fca4c8915182902a01a01e8a8c5bc751)

