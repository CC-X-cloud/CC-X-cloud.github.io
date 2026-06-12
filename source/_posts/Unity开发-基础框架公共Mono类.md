---
title: Unity开发-基础框架公共Mono类
date: 2026-06-10 16:51:03
tags:
  - Unity开发
categories: 编程
cover: https://raw.githubusercontent.com/CC-X-cloud/CC-X-cloud.github.io/refs/heads/main/source/_data/covers/GGMO.webp
---
# 公共Mono模块介绍
公共Mono模块属于基础设施层次，可以让不继承Monobehavior的脚本也可以使用Unity内部的生命周期函数和协程。
公共Mono模块由MonoController和MonoManager两个脚本构成，MonoConrorller实现基础功能，MonoManager为单例模式提供唯一的访问点。
# 公共Mono模块代码实现
Mono模块代码比较简单本质上就是对Unity内部的生命周期函数和协程进行封装，涉及到了代理模式。（后续添加代理模式讲解）
{% codeblock lang:csharp %} 
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MonoController : MonoBehaviour
{
    //给上层Manager提供方法
    private event System.Action updateEvent;

    private void Update()
    {
            updateEvent?.Invoke();    
    }

    public void AddUpdateEvent(Action callback)
    {
        updateEvent += callback;
    }
    public void RemoveUpdateEvent(Action callback)
    {
        updateEvent -= callback;
    }   
    public new void StartCoroutine(IEnumerator routine)
    {
        base.StartCoroutine(routine);
    }
}

{% endcodeblock %}

{% codeblock lang:csharp %} 
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

//单例模式供外部访问
public class MonoManager : Singleton<MonoManager>
{
    private MonoController controller;
    public MonoManager()
    {
        GameObject obj = new GameObject("MonoController");
        controller = obj.GetComponent<MonoController>();
    }
    //对Controller的方法封装
    public void AddUpdateEvent(Action callback)
    {
        controller.AddUpdateEvent(callback);
    }
    public void RemoveUpdateEvent(Action callback)
    {
        controller.RemoveUpdateEvent(callback);
    }
    public new void StartCoroutine(IEnumerator routine)
    {
        controller.StartCoroutine(routine);
    }
}
{% endcodeblock %}