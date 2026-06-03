---
title: Unity游戏开发—角色移动实战Character案例
date: 2026-06-02 20:55:20
tags:
  - Unity开发
categories: 编程
---
# 要求（策划）
**设计意图**
为了解决传统Rigidbody控制器常见的“边缘掉落无法起跳（脑溢血操作）”、“起跳动作与物理位移脱节（滑步感）”以及“斜坡悬空”等痛点，本模块引入了**土狼时间**、**动画事件驱动**与**贴地微重力**机制，旨在打造具有“吸附感”和“高容错”的动作游戏3C体验。
##### 3.2.1 核心机制与边界条件
**① 土狼时间 (Coyote Time / 地面缓冲)**
- **规则**：当角色主动走下悬崖或离开平台边缘时，系统不立即切断“地面状态”，而是保留一个短暂的宽限期。
- **边界条件**：
    - 宽限期内（默认0.1s）按下跳跃，**视为地面起跳**，正常触发跳跃流程。
    - 若角色是**被击飞**或**获得向上初速度**（`verticalVelocity > 0.1`），则**立即强制切断**土狼时间，防止二次起跳。
    - 宽限期结束后未起跳，角色进入下落（Fall）状态。
**② 动画事件驱动起跳 (Anim-Event Driven Jump)**
- **规则**：按下跳跃键不直接赋予物理速度，而是先触发 `Jump` Trigger 播放前摇动画。
- **执行流**：输入跳跃 -> 播放前摇 -> **动画播放到“发力帧”** -> 触发代码 `OnPlayerJump()` -> 赋予真实物理初速度 -> 角色腾空。
- **限制**：起跳后进入 `jumpCooldown` (0.5s)，期间屏蔽所有跳跃输入。
**③ 贴地微重力 (Sticky Gravity)**
- **规则**：角色处于地面状态（`isGrounded == true`）且未起跳时，不使用 `0` 作为垂直速度，而是强制施加 `-2 m/s` 的向下微重力。
- **目的**：确保角色走下斜坡、跨越微小台阶时，胶囊体能紧紧“吸”住地面，杜绝物理引擎计算延迟导致的“下坡悬空”或“落地弹跳”现象。
##### 3.2.2 策划配置表 (Inspector 暴露参数)

|参数名 (代码变量)|类型|默认值|调参指导与边界限制|
|---|---|---|---|
|`jumpHeight`|Float|`1.5`|**跳跃绝对高度(米)**。程序已做公式反推，改此值即可，**勿直接改初速度**。|
|`jumpCooldown`|Float|`0.5`|跳跃硬直冷却(秒)。若需实现“连跳”流派，需通过Buff将此值降为0。|
|`groundBufferTime`|Float|`0.1`|**土狼时间(秒)**。建议范围 `0.08 ~ 0.15`。>0.2s会导致明显的“空中二段跳”视觉Bug。|
|`groundCheckDistance`|Float|`0.2`|脚底射线长度(米)。**注意**：修改角色胶囊体高度/半径时，需同步微调此值。|
|`gravity`|Float|`-9.81`|基础重力。若Boss战需要“厚重感”，可通过区域Trigger将此值临时改为 `-15.0`。|

可以看看根据这些要求能不能写出来
# 需求分析和代码实现
从策划文字到落地中间有很大的挑战，这里我用加法原则，先保证基础功能，在往里做加法。
比如这里就可以先写基础的WSAD移动实现，然后加入简单跳跃功能，然后后优化跳跃做出地面缓冲，加入动画控制等。
1.基础平面控制WSAD控制逻辑
2.跳跃控制（需要自己实现重力和补足地面检查）先实现简单的跳跃。
3.加入动画控制
4.优化跳跃控制加入地面缓冲，跳跃冷却，跳跃中检测。
{% codeblock [title] [lang:Shaderlap] [url] [link text] [additional options] %}
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerMovement : MonoBehaviour
{
    [Header("MovementSpeed")]
    [SerializeField] private float movementSpeed = 3f;
    [Header("RotationSpeed")]
    [SerializeField] private float rotationSpeed = 5f;
    [Header("Jump")]
    [SerializeField] private float jumpCooldown = 0.5f;
    [SerializeField] private float jumpHeight = 1.5f;
    [Header("Camera")]
    [SerializeField] private Transform cameraTransform;
    [Header("Gravity")]
    [SerializeField] private float gravity = -9.81f;
    [Header("Ground Check")]
    [SerializeField]private float groundCheckDistance = 0.2f;
    [SerializeField]private LayerMask groundMask;
    [SerializeField]private float groundBufferTime = 0.1f;

    private float verticalVelocity = 0;
    private float factor  = 1f;
    private float lastJumpTime = -10f;
    private float lastGroundedTime;

    //角色身上的组件引用
    private Rigidbody rb;
    private Animator animator;
    private CharacterController characterController;

    private bool rawIsGrounded;
    private bool isGrounded;
    private bool isJumping;
    private bool hasLeftGround = false;

	//事件总线系统，增强解耦性
    private void OnEnable()
    {
        GameEventMgr.Instance.Subscribe<PlayerMoveEvent>(HandlePlayerMove);
        GameEventMgr.Instance.Subscribe<PlayerJumpEvent>(PlayerJump);
    }
    private void OnDisable()
    {
        GameEventMgr.Instance.UnSubscribe<PlayerMoveEvent>(HandlePlayerMove);
        GameEventMgr.Instance.UnSubscribe<PlayerJumpEvent>(PlayerJump);
    }

    void Start()
    {
        //初始化
        rb = GetComponent<Rigidbody>();
        animator = GetComponent<Animator>();
        characterController = GetComponent<CharacterController>();

        animator.applyRootMotion = false;
        rb.isKinematic = true;
        rb.useGravity = false;
        rb.constraints = RigidbodyConstraints.FreezeRotation;

        
    }
    void Update()
    {
        CheckGrounded();
        useGravity();
        UpdateAnimation();

    }

	//优化地面检测
    private void CheckGrounded()
    {
        bool ccGrounded = characterController.isGrounded;

        float rayOrignY = characterController.bounds.min.y + 0.05f;
        Vector3 rayOrigin = new Vector3(transform.position.x, rayOrignY, transform.position.z);
        bool rayGrounded = Physics.Raycast(rayOrigin, Vector3.down, out RaycastHit hit,
                                            groundCheckDistance,groundMask);

        rawIsGrounded = ccGrounded || rayGrounded;

        if(rawIsGrounded)
        {
            lastGroundedTime = Time.time;
        }

        if (!rawIsGrounded && (Time.time - lastGroundedTime) <= groundBufferTime)
        {
            isGrounded = true;
        }
        else
        {
            isGrounded = rawIsGrounded;
        }

        if (verticalVelocity > 0.1f)
        {
            isGrounded = false;
        }
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
	//跳跃主要逻辑
    public void PlayerJump(PlayerJumpEvent eventData)
    {
        if(Time.time - lastJumpTime < jumpCooldown)
        {
            return; //跳跃冷却中，不能跳跃
        }
        if (isGrounded && !isJumping)
        {
            isJumping = true;
            hasLeftGround = false;
            lastJumpTime = Time.time;
            animator.SetTrigger("Jump");

        }
    }
    //使用动画事件来应用翻译
    public void OnPlayerJump()
    {
        if (isJumping)
        {
            verticalVelocity = Mathf.Sqrt(jumpHeight * -2f * gravity);
            hasLeftGround = true;
        }
    }
    //模拟重力补足CharacterControl的缺陷
    public void useGravity()
    {
        verticalVelocity += gravity * Time.deltaTime;
        characterController.Move(new Vector3(0, verticalVelocity, 0) * Time.deltaTime);
        if (isGrounded && hasLeftGround)
        {
            verticalVelocity = -2f; //保持角色贴地
            isJumping = false;    // 真正落地，结束跳跃流程
            hasLeftGround = false;
        }
        else if(isGrounded && verticalVelocity < -2)
        {
            verticalVelocity = -2f; //没有起跳但落地时保持角色贴地
        }
    }
	//动画更新
    public void UpdateAnimation()
    {
        bool isWalking = Input.GetKey(KeyCode.W) || Input.GetKey(KeyCode.A) || Input.GetKey(KeyCode.S) || Input.GetKey(KeyCode.D);
        animator.SetBool("IsGrounded", isGrounded);
        animator.SetBool("IsWalking", isWalking);
        if (Input.GetKey(KeyCode.LeftShift) && isWalking)
        {
            animator.SetBool("IsRunning", true);
        }
        else
        {
            animator.SetBool("IsRunning", false);
        }
    }
}
{% endcodeblock %}
# 后续可优化内容（不定时更新）
1.根据项目需求加入更多的动画