---
title: Unity游戏开发-基础设施单例基类
date: 2026-06-09 19:46:57
tags:
  - Unity开发
categories: 编程
cover: https://raw.githubusercontent.com/CC-X-cloud/CC-X-cloud.github.io/refs/heads/main/source/_data/covers/JCDLJL.webp
---
# 设计模式单件模式
其简洁的定义为单件模式确保一个类只有一个实例，并提供一个全局访问点。
它可以避免访问混乱，资源浪费和性能损耗，避免状态冲突与数据不一致，避免重复操作与资源竞争。
# Unity游戏开发的单件模式设计：单例基类

单例基类属于游戏框架时的基础设施，为核心服务层提提供统一父类，避免重复书写单例模式。
这里我们将单例基类分为三种：继承自Unity的Monobehavior，纯C#，特殊的单例基类。
针对前两种单例基类还可以被写为带有以下优化特性：双重检查锁定，泛型，持久化，调节器，空对象。
其中双重检查锁定，泛型，空对象是两种单例都可以用，而持久化，调节器只有继承自Unity的Monobehavior
## 纯C#的单例基类
单例不同的实例化时机可以被叫做**饿汉式**和**懒汉式**。
1. **饿汉式（Eager Initialization）**：在类被加载时就直接创建好实例。优点是简单且天然线程安全；缺点是即使不使用该实例也会占用内存资源。
2. **懒汉式（Lazy Initialization）**：在第一次真正需要使用该实例时才进行创建。优点是延迟加载、节省内存；缺点是如果不加锁，在多线程环境下可能会出现线程安全问题。
还有利用C#提供的`Lazy<T>`类来实现的，由框架底层保证线程安全和延迟加载，代码更加简洁优雅。
## 继承自Unity的Monobehavior
不存在饿汉式的单例模式，因为Unity**不允许通过 `new` 关键字来实例化继承自 `MonoBehaviour` 的类**。`MonoBehaviour` 必须依附于游戏物体（GameObject），并通过 `AddComponent` 的方式添加到场景中才能生效。
## 特殊的单例基类
在大型现代游戏架构中，为了降低单例带来的强耦合，常采用以下方案替代传统的单例：
1. **ScriptableObject 单例**：利用 Unity 的 ScriptableObject 特性，将配置数据作为资产（Asset）保存在项目中，通过 `Resources.Load` 或 Addressables 全局读取，非常适合做游戏设置和持久化数据管理。
2. **服务定位器模式（Service Locator）**：提供一个全局的注册与获取中心，比单例更灵活，便于在运行时动态替换服务实现，也更容易进行单元测试
# 代码实现
提供的代码可以经过删改来得到实际开发中需要的代码。
## 最基础式C#纯代码

{% codeblock [lang:CS] %}
public class SingleCS
{
    //懒汉式
    private static SingleCS instance;
    private SingleCS() { }
    public static SingleCS Instance
    {
        get
        {
            if (instance == null)
            {

                instance = new SingleCS();
            }
            return instance;
        }
    }
    //饿汉式
    //private static readonly SingleCS instance2 = new SingleCS();
    //public static SingleCS Instance2 => instance2;
    //private SingleCS() { }
}

{% endcodeblock %}
## C#纯代码
添加了多种优化方式可以根据实际要求修改
{% codeblock [lang:CS] %}
using UnityEngine;
/// <summary>
/// 纯C#单例基类
/// </summary>
/// <typeparam name="T">继承此基类的具体类</typeparam>
public abstract class PureSingleton<T> where T : class, new()
{
	private static T _instance;
	private static readonly object _lock = new object();
	/// <summary>
	/// 全局访问点（线程安全 + 双重检查锁定）
	/// </summary>
	public static T Instance
	{
		get{
		// 第一次检查（无锁，快速路径）
		if (_instance == null){
		lock (_lock)
		{
		// 第二次检查（加锁，防止多线程重复创建）
		if (_instance == null)
			{
		_instance = new T();
			}
		}
	}
		return _instance;
	}}
	/// <summary>
	/// 安全获取实例（空对象模式）
	/// 当实例为null时，不会抛出异常，而是返回一个默认的空实现
	/// 注意：由于泛型约束限制，这里提供扩展方法或在具体子类中实现更优雅
	/// </summary>
	public static T SafeInstance => _instance ?? Instance;
	/// <summary>
	/// 显式销毁单例（纯C#不会随场景销毁，需手动清理）
	/// </summary>
	public static void DestroyInstance()
	{
		lock (_lock){
		if (_instance != null)
		{
		// 如果子类实现了 IDisposable 或自定义清理接口，可在此调用
		_instance = null;
		}
	}
}
{% endcodeblock %}
## 继承Monobehavior的单例基类
{% codeblock [lang:Shaderlap] %}
using UnityEngine;
/// <summary>
/// 继承MonoBehaviour的单例基类
/// </summary>
/// <typeparam name="T">继承此基类的具体MonoBehaviour类</typeparam>
public abstract class MonoSingleton<T> : MonoBehaviour where T : MonoSingleton<T>
{
    private static T _instance;
    private static readonly object _lock = new object();
    private static bool _applicationIsQuitting = false;

    /// <summary>
    /// 全局访问点（懒汉式 + 自动创建 + 退出保护）
    /// </summary>
    public static T Instance
    {
        get
        {
            // 1. 应用退出保护：防止在OnDestroy中访问单例导致Unity报错
            if (_applicationIsQuitting)
            {
                Debug.LogWarning($"[MonoSingleton] 实例 '{typeof(T)}' 在应用退出时被访问，返回 null。");
                return null;
            }

            lock (_lock)
            {
                if (_instance == null)
                {
                    // 2. 调节器机制：尝试在场景中查找（防止手动拖拽了预制体但Instance未赋值）
                    _instance = FindObjectOfType<T>();

                    // 3. 如果场景中不存在，自动创建（懒汉式）
                    if (_instance == null)
                    {
                        GameObject singletonObject = new GameObject(typeof(T).Name);
                        _instance = singletonObject.AddComponent<T>();
                    }
                }
                return _instance;
            }
        }
    }
    /// <summary>
    /// 核心生命周期：保证唯一性与持久化
    /// </summary>
    protected virtual void Awake()
    {
        // 调节器机制：如果场景中存在多个实例，销毁多余的
        if (_instance == null)
        {
            _instance = this as T;
            
            // 持久化特性：跨场景切换时不销毁
            if (Application.isPlaying)
            {
                DontDestroyOnLoad(gameObject);
            }
        }
        else if (_instance != this)
        {
            Debug.LogWarning($"[MonoSingleton] 发现重复实例 '{typeof(T)}'，正在销毁多余对象。");
            Destroy(gameObject);
        }
    }
    /// <summary>
    /// 监听应用退出事件
    /// </summary>
    protected virtual void OnApplicationQuit()
    {
        _applicationIsQuitting = true;
    }
    /// <summary>
    /// 销毁时清理静态引用
    /// </summary>
    protected virtual void OnDestroy()
    {
        if (_instance == this)
        {
            _instance = null;
        }
    }
}
{% endcodeblock %}
## 特殊的单例基类
{% codeblock [lang:Shaderlap] %}
using UnityEngine;
/// <summary>
/// 特殊的单例基类：基于 ScriptableObject 的配置单例
/// 结合了数据与代码的持久化，非常适合做全局配置
/// </summary>
/// <typeparam name="T">继承此基类的 ScriptableObject 类</typeparam>
public abstract class PersistentConfigSingleton<T> : ScriptableObject where T : PersistentConfigSingleton<T>
{
    private static T _instance;

    /// <summary>
    /// 全局访问点（从Resources或Addressables中懒加载）
    /// </summary>
    public static T Instance
    {
        get
        {
            if (_instance == null)
            {
                // 约定俗成：将配置资产放在 Resources/Configs 目录下
                // 实际项目中可替换为 Addressables.LoadAssetAsync<T>
                _instance = Resources.Load<T>($"Configs/{typeof(T).Name}");
                
                if (_instance == null)
                {
                    Debug.LogError($"[ConfigSingleton] 未在 Resources/Configs 目录下找到配置: {typeof(T).Name}");
                }
            }
            return _instance;
        }
    }

    /// <summary>
    /// 编辑器下或热重载时手动重置单例引用
    /// </summary>
    [ContextMenu("Reset Instance")]
    private void ResetInstance()
    {
        _instance = null;
    }
}
{% endcodeblock %}