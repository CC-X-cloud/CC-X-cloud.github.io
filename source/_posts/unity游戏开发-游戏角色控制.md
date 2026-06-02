---
title: unity游戏开发-游戏角色控制
date: 2026-05-31 14:45:01
tags:
  - Unity开发
categories: 编程
cover: https://raw.githubusercontent.com/CC-X-cloud/CC-X-cloud.github.io/refs/heads/main/source/_data/covers/MoveFF1.webp
---
# 移动控制的方案分类
按照是否受到物理影响分为**非物理驱动**和**物理驱动**还有就是**混合方案**。
## 非物理驱动（代码绝对控制）
这类方法不依赖刚体物理，直接移动物体。适用于没有真实物理交互需求，或需要绝对控制权的游戏，其最主要的方案为Transform和CharacterController。
### 1.Transform
- **原理**：每帧直接修改 `transform.position` 或调用 `transform.Translate()`。
- **核心特征**：
    - **零物理开销**：这是性能最极致的方式。
    - **绝对精准**：你设什么位置就是什么位置，没有惯性、没有重力、没有任何延迟。
    - **完全无碰撞**：会直接穿透任何物体，除非你自己写代码处理。
- **典型用途**：
    - UI 元素移动。
    - 过场动画中无需物理的对象。
    - 需要精确控制的机关（如《传送门》里的光桥）
### 2.Character Controller
- **原理**：这是 Unity 封装好的一个高级组件，核心是一个胶囊体碰撞器。你调用 `Move()` 方法，它会在内部每帧“先移动胶囊体，检测碰撞，再回退解决穿透”。
- **核心特征**：
    - **开箱即用的角色功能**：自带处理**楼梯（Step Offset）** 和 **斜坡（Slope Limit）** 的能力，这是它区别于纯物理方案的最大优势。
    - **非物理体**：它不遵循牛顿力学。它不会被其他刚体推动，你要推它只能自己写脚本。
    - **控制精准**：与 Transform 一样，你说走多远就走多远，但会被墙壁拦住，并且会自然“滑过”低于 Step Offset 的障碍物。
- **本质理解**：它是一个**带有高级碰撞解决算法的运动学控制器**。
- **官方API参考**：[CharacterController - Unity 脚本 API --- CharacterController - Unity 脚本 API](https://docs.unity.cn/cn/2022.3/ScriptReference/CharacterController.html)
### 3. NavMesh Agent（导航网格代理）
- **原理**：挂载在角色上，配合 Unity 的导航系统。你给它一个目标点 `SetDestination()`，它会自动寻路并移动。
- **核心特征**：
    - **自动寻路**：这是它的核心价值，内部使用 A* 算法。
    - **自动避障**：多个 Agent 之间可以互相躲避（需要开启避障）。
    - **本质仍是角色控制器**：它的底层移动和碰撞解决机制与 Character Controller 非常相似。
- **典型用途**：AI 敌人巡逻、NPC 移动。几乎不用于主控玩家角色。
## 物理驱动（ 控制权交给物理引擎）
这类方法把移动完全交由物理引擎模拟。
官方API参考：[Rigidbody - Unity 脚本 API --- Rigidbody - Unity 脚本 API](https://docs.unity.cn/cn/2022.3/ScriptReference/Rigidbody.html)
### 1. Rigidbody 动力学
- **原理**：通过 `AddForce` 或直接设置 `velocity` 来驱动一个非运动学（`isKinematic = false`）的刚体。
- **核心特征**：
    - **真实的物理模拟**：自然拥有重力、摩擦力、反弹和惯性。
    - **完全交互**：会与场景中所有物理对象发生碰撞、推动和被推动。
    - **控制感弱**：惯性导致转向不灵敏，斜坡容易滑落。**直接设置 `velocity` 是唯一能获得精确控制感的方法**，但本质仍是物理模拟。
- **本质理解**：你是对角色**施加一个“请求”**，最终的运动由物理引擎根据力和质量算出。
### 2. Rigidbody 运动学

- **原理**：勾选 `isKinematic = true`，然后用 `MovePosition` 和 `MoveRotation` 方法在 `FixedUpdate` 中移动。
- **核心特征**：
    - **推土机**：它是物理世界里“说一不二”的权威。它可以推开挡路的动力学刚体，自己却纹丝不动。
    - **无物理模拟**：不受重力、推力影响，完全由你的脚本控制轨迹。
    - **碰撞检测有效**：它仍会与静态碰撞体和动力学刚体发生碰撞。
- **典型用途**：移动平台、电梯、缓缓打开的巨大石门。
## 混合方案
实践中，高质量的角色控制往往是上述方法的组合，本质可以算做物理驱动的子类。
### 1. Character Controller + 运动学 Rigidbody（“大使馆模式”）
- **分工**：
    - **Character Controller**：负责角色的移动、爬坡、卡墙解决。
    - **运动学 Rigidbody**：充当角色在物理世界的“外交官”。它自己不动（因为 CC 直接移动了 `transform`），但其他物理物体“认识”它。
- **好处**：你可以用脚本获取碰撞信息（如 `OnControllerColliderHit`），然后通过被碰物体的 `Rigidbody.AddForce()` 来手动实现“踢飞杂物”。外部的物理冲击（如爆炸）也能通过代码获取，并手动转换给 CC 的 `Move()`。
### 2. 纯手动碰撞检测
- **原理**：角色移动用 `Transform`，但在移动前，用射线或 `Physics.BoxCast` 检测前方有无障碍物。
- **核心特征**：
    - **完全自定义**：你可以实现物理引擎做不到的移动逻辑，比如“在墙上行走”或“穿过特定类型的物体”。
    - **开发成本高**：你要处理所有情况：斜坡、台阶、与多个面的挤压、被推动的物体等。
- **典型用途**：对物理要求极其特殊的游戏，或早期原型快速实现独特想法。不推荐作为通用方案。

### 3. 有限状态机混合（高级方法，小项目一般用不到）
这是最实用、最灵活的生产方式。根据角色所处的不同状态，切换不同的驱动方式。

|角色状态|使用的驱动方式|说明|
|---|---|---|
|**常规移动**|Character Controller (或 CC + 运动学刚体)|享受精准操控和内置斜坡台阶处理。|
|**死亡/击飞**|纯非运动学 Rigidbody (布娃娃)|身体瘫软、被炸飞，完全交给物理模拟。需要动画师提前准备。|
|**攀爬/翻越**|Transform 直接位移|检测到齐腰高障碍，将 CC 禁用，用动画和代码控制角色精确位移到翻越点，再切回 CC。|
|**过场/固定轨迹**|运动学 Rigidbody + 动画驱动|角色移动完全由动画驱动，但身体能与物理世界互动。|
## 对比表格
不代表全部的方案，只列举目前我自己知道的方法

|特性|Transform 直接位移|Character Controller|NavMesh Agent|Rigidbody (动力学)|Rigidbody (运动学)|CC + 运动学 Rigidbody|
|---|---|---|---|---|---|---|
|**物理引擎参与**|无|仅碰撞检测|仅碰撞检测|完全模拟|仅碰撞检测|仅碰撞检测|
|**碰撞响应**|无，直接穿透|有，被挡住并滑动|有，被挡住并滑动|有，真实反弹|有，但自身不反弹|CC 挡，物理体识别它|
|**重力**|无，需自己写|内置 (SimpleMove)|内置|内置|无|用 CC 的|
|**惯性**|无|无|无|有|无|无|
|**爬坡/上台阶**|无，需自己写|✅ 内置|✅ 内置|❌ 需自己处理|❌ 需自己处理|✅ 用 CC 的|
|**被外力推动**|不能|不能 (需代码模拟)|不能|✅ 自然响应|不能|不能 (可代码获取力)|
|**推动其他物体**|不能|不能 (需代码检测)|不能|✅ 自然推动|✅ 能推开动力学刚体|可代码实现踢飞|
|**移动方式**|直接设 position|Move() / SimpleMove()|SetDestination()|AddForce() / velocity|MovePosition()|用 CC 的 Move()|
|**性能开销**|极低|中|中高 (含寻路)|高|中|中|
|**控制精度**|绝对精确|精确|目标导向|低，有惯性延迟|绝对精确|精确|
|**适合主角**|❌|✅ 最常用|❌ (AI用)|仅限特殊类型|❌|✅ 高级方案|
|**适合 AI/NPC**|❌|✅|✅ 推荐|❌|❌|✅|
|**适合载具**|❌|❌|❌|✅|❌|❌|
|**适合移动平台**|❌|❌|❌|❌|✅ 最佳|❌|
# 移动控制的三个基础方法
##  1.Transform 直接位移
最直接的方式：通过脚本直接修改 transform.position 或 transform.Translate()。
- **优点**
    - **性能最好**：完全绕过物理引擎。
    - **代码极简**：想移动到哪就设哪，适合实现闪现、瞬移。
    - **结果绝对可控**：不会有惯性、重力干扰，适合做无缝传送或精确的过场动画。
- **缺点**
    - **完全无碰撞**：会直接穿透墙壁和其他物体。如果强行用它实现“伪碰撞”，需要自己写大量射线检测，费时又容易出错。
    - **无物理反馈**：其他物理对象不会对它做出反应。
    - **帧率敏感**：直接设位置时，如果不乘以 `Time.deltaTime`，在不同帧率下移动速度会不同。
**最适合**：UI动画、无交互的环境漫游、过场镜头。
## 2. Character Controller
这其实是一个**专为角色移动设计的“胶囊形”碰撞体**，不是刚体。它自带碰撞检测，但不遵循物理定律。
- **优点**
    - **内置角色功能**：能自动处理楼梯和坡度（设置 `Step Offset` 和 `Slope Limit`），比用刚体做角色省心得多。
    - **移动后碰撞不穿透**：调用 `Move()` 后，它会自动解决穿透，让你靠在墙边或地面，且不会被其他刚体轻易推动。
    - **控制力强**：保留了精确的位置控制，没有惯性，适合响应灵敏的动作游戏。
- **缺点**
    - **不是物理体**：它不会受力，不会被爆炸炸飞，除非你自己写代码模拟。
    - **运动受限**：只能用 `Move()` 和 `SimpleMove()` 来移动，旋转也要手动写在代码里，无法像刚体那样旋转着撞开。
    - **对高速物体仍会穿透**：如果角色或别的物体移动过快，仍可能漏掉碰撞检测。需要配合 `Rigidbody` 的连续碰撞检测来缓解。

**最适合**：**绝大多数第一/第三人称游戏的主角**。这些角色需要被精确控制，又能自然地爬坡、下楼梯。
## 3. Rigidbody 刚体
这是完整的物理模拟对象，受重力和作用力影响。
- **优点**
    - **最真实的物理**：自然拥有重力、摩擦力、反弹。推箱子、汽车、翻滚的足球，用它准没错。
    - **完整的交互性**：能自然地与其他刚体互相推动，或被墙挡住。使用物理材质就能轻松做出滑冰、弹跳的效果。
    - **丰富的内置功能**：有重力、阻力、速度锁定等，不用自己写。
- **缺点**
    - **控制难、手感肉**：惯性会带来明显延迟，角色不会说停就停。实现灵敏、紧凑的移动手感需要精细调整参数，或直接覆写 `velocity`。
    - **可能“抽风”**：如果被卡在几个碰撞体之间，会产生剧烈抖动甚至弹飞，需要代码处理。
    - **性能开销大**：物理模拟计算比前两者都消耗性能。
**移动时的关键区别**：
- **`AddForce()`**：模拟真实受力，有加速过程。适用于物理驱动（推箱子）。
- **直接设置 `velocity`**：人为设定速度，**绕过惯性直接达到目标速度**。适合对控制精度有要求、但仍需物理交互的角色（如平台跳跃）。
**最适合**：载具、足球、弹跳球，或任何需要被炸飞、推开的对象。平台跳跃类角色也常用，方便实现推箱子。
## 对比表格
|特性|Transform 直接位移|Character Controller|Rigidbody 动力学|Rigidbody 运动学|
|---|---|---|---|---|
|**物理交互**|无，会直接穿模|有限，能碰撞但不会被推动|完整，真实的牛顿力学|单向，能推别人，自己不被推|
|**性能开销**|极低|中等|高|中等|
|**控制精度**|绝对精确，指哪走哪|精确，但受碰撞约束|低，有惯性、反弹等|绝对精确，指哪走哪|
|**典型用途**|UI元素、简单移动、过场动画|玩家角色、AI移动|足球、赛车、需要被炸飞的物体|移动平台、电梯、机关门|
|**移动方式**|直接修改 position|使用 Move() / SimpleMove()|使用 AddForce() 或直接设 velocity|使用 MovePosition() / MoveRotation()|

# 基础方法的代码实现讲解
## 1.Transform 直接位移
{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class TransformMove : MonoBehaviour
{
    [Header("Transform Speed")]
    [SerializeField] private float Speed = 5f;

    //声明变量接收AD和WS方向上是输入
    private float HorizontalInput;
    private float VerticalInput;

    void Update()
    {
        //Translate方法
        //获得输入数据
        HorizontalInput = Input.GetAxis("Horizontal");
        VerticalInput = Input.GetAxis("Vertical");

        //根据向量的合成可以得出移动方向的向量
        Vector3 MoveVector = new Vector3 (HorizontalInput, 0 , VerticalInput);
        MoveVector.Normalize();
        //乘以deltaTime 让移动按秒为单位，而不是帧
        MoveVector = MoveVector * Speed * Time.deltaTime;
        transform.Translate(MoveVector);

        //更新position的方法
        //用了点加向量等于点的数学方法。
        //Vector3 newPos = transform.position + MoveVector;
        //transform.position = newPos;

    }
}
{% endcodeblock %}
## 2.Character Controller
在这个脚本中原生提供了两种方法，主要区别就是Move没有重力向量的要乘deltatime，而SimpleMove有重力不用乘deltatime
{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class CharacterControllerMove : MonoBehaviour
{
    [Header("CharacterController Speed")]
    [SerializeField] private float Speed = 5f;

    //提供脚本容器以供调用API
    private CharacterController cc;

    //获得方向输入的容器
    private  float HorizontalInput;
    private float VerticalInput;

    private void Start()
    {
        //初始化脚本信息，获得脚本
        cc = GetComponent<CharacterController>();
    }
    void Update()
    {
        //获得输入值
        HorizontalInput = Input.GetAxis("Horizontal");
        VerticalInput = Input.GetAxis("Vertical");

        //计算移动的向量
        Vector3 MoveVector = new Vector3(HorizontalInput, 0, VerticalInput);
        MoveVector.Normalize();
        MoveVector = MoveVector * Speed * Time.deltaTime;
        //调用API
        cc.Move(MoveVector);

        //SimpleMove方法
        //MoveVector = MoveVector * Speed;
        //cc.SimpleMove(MoveVector);


    }
}
{% endcodeblock %}
## 3.rigidbody方法
刚体的移动方案有很多，这里之讲最基础和最常用的3种分别是MovePosition和修改Velocity使用AddForce。
### 对比表格
| 方案               | 基础介绍                   | 核心特性                                                      | 选择依据                                    |
| ---------------- | ---------------------- | --------------------------------------------------------- | --------------------------------------- |
| **MovePosition** | 直接设置刚体的目标位置，每帧"传送"到新位置 | • 瞬间响应，无延迟  <br>• 不受摩擦力和阻力影响  <br>• 控制精度最高  <br>• 无需保留重力  | ✅ 需要精准操控  <br>✅ 希望按键即动  <br>✅ RPG、FPS游戏 |
| **Velocity**     | 改变刚体的速度向量，让物理引擎控制移动    | • 有惯性感，会滑行  <br>• 受地面摩擦影响  <br>• 需要手动保留Y轴重力  <br>• 碰撞反应真实 | ✅ 想要真实惯性  <br>✅ 需要上坡减速  <br>✅ 赛车、平台跳跃   |
| **AddForce**     | 给刚体施加力，由物理引擎计算加速度      | • 最真实的物理效果  <br>• 质量影响移动（F=ma）  <br>• 有加速度累积  <br>• 控制感最弱 | ✅ 追求物理真实  <br>✅ 需要力的累积  <br>✅ 解谜、太空游戏   |

{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class rigbodyMove : MonoBehaviour
{
    [Header("RigidBody Move")]
    [SerializeField] private float Speed = 5f;
    //加力方式，默认是连续加力，如果想要瞬间加力，可以改成Impulse
    //[SerializeField] private ForceMode forceMode = ForceMode.Force;

    private Rigidbody rb;

    private float HorizontalInput;
    private float VerticalInput;
    void Start()
    {
        rb = GetComponent<Rigidbody>();
    }

    //物理驱动方法要用物理帧更新，物理更新时间间隔固定，和帧率无关。
    void FixedUpdate()
    {
        HorizontalInput = Input.GetAxis("Horizontal");
        VerticalInput = Input.GetAxis("Vertical");

        //方案一
        Vector3 MoveVector = new Vector3(HorizontalInput, 0, VerticalInput); 
        MoveVector.Normalize();
        Vector3 NewPosition = transform.position + MoveVector * Speed * Time.fixedDeltaTime;
        rb.MovePosition(NewPosition);
        //方案二
        //Vector3 MoveVector = new Vector3(HorizontalInput, 0, VerticalInput);
        //MoveVector.Normalize();
        //MoveVector *= Speed;
        //rb.velocity = MoveVector;
        //方案三
        //Vector3 MoveVector = new Vector3(HorizontalInput, 0, VerticalInput);
        //MoveVector.Normalize();
        //MoveVector = MoveVector * Speed * Time.fixedDeltaTime;
        //rb.AddForce(MoveVector, forceMode);

    }
}
{% endcodeblock %}