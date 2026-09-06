# CAPABILITY-13：能力认证与异步人机共识（HITL）规范（v1.0.0-RFC-4）
> © 2026 CommonIntents. 依据 CC BY-ND 4.0 许可（https://creativecommons.org/licenses/by-nd/4.0/）。

## 1. 引言与设计目标
本规范定义了 **CAPABILITY-13** 标准，它是 **CommonIntents-144（CI-144）** 协议族中面向AI原生的能力认证、动态安全域映射与异步非阻塞人机共识（HITL）标准。

在多智能体共生的复杂环境中，编译期静态安全控制会导致系统僵化，而无约束的自主执行则可能引发灾难性载荷执行或密钥泄露。CAPABILITY-13 通过以下机制解决该问题：
- 定义基于令牌的异步质询-响应协议完成人机校验，避免有限状态机线程阻塞与资源耗尽
- 建立可动态更新、经密码学签名的权限映射注册表（`capability_mapping.toml`）
- 标准化逻辑意图作用域与物理执行沙箱权限的直接边界映射（如 Tentacle 权限清单）

---

## 2. 动态权限映射注册表
CAPABILITY-13 不将安全映射硬编码在执行器（Anaphase）的编译产物中，而是将权限判定委托给动态注册表文件 `capability_mapping.toml`。

### 2.1 结构定义
```toml
# config/capability_mapping.toml
# 经 Ed25519 签名的动态权限映射配置

[meta]
version = "1.0.0"
updated_at = "2026-06-26T12:00:00Z"
signer_l0_hash = "f3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca4..." # 创建者L0创世密钥哈希

[mappings.standard_scopes]
"read" = ["filesystem:read", "memory:read"]
"write" = ["filesystem:write", "memory:write"]
"execute" = ["execute:wasm", "execute:js"]
"external_network" = ["network:outbound"]

[mappings.custom_scopes]
"media_control" = ["system:audio", "execute:media_player"]
"scientific_simulation" = ["compute:float_matrix", "filesystem:read:/opt/data"]

[integrity]
algorithm = "Ed25519"
# 覆盖全部 mappings 区块的密码学签名，由创建者私钥签发
signature = "8b625cf0a359873d2efba93de89fa1237de1e604f378d389fe39a1c890bfca12f9..."
```

### 2.2 启动密码学校验链路
执行器（Anaphase）初始化阶段必须执行类 DNSSEC 的签名校验流程：
1.  **提取映射区块**：读取配置文件中所有 `[mappings]` 区块的原始字节数组
2.  **解析创建者公钥**：通过本地 UDS 上的标准化 L0 gRPC 接口，向系统受信的 **L0 控制器** 查询与当前 L0 基因锁谱系哈希匹配的受信创建者公钥
3.  **解密验签**：使用提取出的映射载荷，校验 `[integrity].signature` 区块内的签名有效性
4.  **失败安全锁定**：若签名校验失败，或 `signer_l0_hash` 与当前 L0 身份不匹配，Anaphase 必须立即抛出错误码 `0x05FA`（配置签名无效），拒绝启动，并全面锁定所有执行沙箱

---

## 3. 异步非阻塞人机质询-响应协议
当高风险操作被拦截或关键权限被调用时，CAPABILITY-13 强制采用非阻塞异步挂起流程，复用 BIND-19 的 `CON`（`0x02`）标志位。

```
                  Anaphase 拦截高风险操作
                                   │
                                   ▼
                [ 步骤1：挂起任务并生成校验令牌 ]
          - 创建待人机确认令牌，绑定关联ID
          - 让出当前任务状态机线程，避免资源阻塞
                                   │
                                   ▼ (令牌流式推送至UI视口)
                [ 步骤2：可视化与物理身份质询 ]
          - 界面展示「需共识确认」视觉锁定状态
          - 等待人类创建者对载荷进行物理签名
                                   │
                                   ▼ (返回已签名的响应令牌)
                [ 步骤3：验签并恢复执行 ]
          - 使用L0密钥校验密码学签名
          - 恢复被挂起的状态机上下文，执行工具，提交两阶段提交
```

### 3.1 待确认人机令牌（Pending_HITL_Token）载荷规范
```json
{
  "correlation_id": "4bf92f35-77b3-4da6-a3ce-929d0e0e4736",
  "challenge_hash": "目标命令与参数的SHA-256哈希值",
  "requested_permissions": ["network:outbound", "filesystem:write"],
  "timeout_ms": 900000,
  "timestamp": 1782376405
}
```

#### 字段说明
- `correlation_id`：可选字符串，用于协议栈全链路追踪与事务关联
- `challenge_hash`：字符串，目标命令载荷的 SHA-256 密码学哈希
- `requested_permissions`：字符串数组，本次操作申请的具体物理权限

### 3.2 非阻塞状态机流转
- **线程让出**：发出待确认令牌后，状态机立即让出当前活跃线程，不会阻塞操作系统线程，也不会进入死循环等待
- **任务调度**：调度器将当前任务切换为「等待中」状态，允许执行队列中其他待处理任务占用CPU资源
- **验签唤醒**：已签名的响应令牌通过 BIND-19 控制帧（0x03）回推后，Anaphase 反应器完成签名校验；校验通过则将任务切回「活跃」状态，无缝恢复执行上下文

---

## 4. 标准能力映射配置
所有执行沙箱（如 Tentacle WASM）必须在其静态权限清单中声明申请的权限。CAPABILITY-13 强制执行以下精细化边界映射：

| 逻辑作用域 | Tentacle 清单能力项 | 沙箱执行行为 |
|:---|:---|:---|
| **`read`（读取）** | `"filesystem:read"` | 授予指定隔离根目录的只读访问权限 |
| **`write`（写入）** | `"filesystem:write"` | 仅允许在临时写时复制沙箱层内执行写入 |
| **`execute`（执行）** | `"execute:wasm"` / `"execute:js"` | 启动受内存与CPU边界限制的沙箱化 WASM/QuickJS 线程 |
| **`external_network`（外网访问）** | `"network:outbound"` | 允许出站TCP连接；所有出站调用必须经本地 Tuck 代理转发 |
| **`admin`（管理）** | `"admin:gene_lock"` | 允许向记忆核心L0控制器发送管理重载指令 |

---

## 5. 创建者密钥轮换与吊销协议
为防范私钥泄露风险，CAPABILITY-13 与 L0 控制器联动，支持类 GPG 的非对称密钥轮换：
1.  **提交轮换申请**：所有者通过管理通道提交已签名的密钥轮换事务：
    ```json
    {
      "previous_key_hash": "当前签名密钥的SHA-256哈希",
      "new_public_key": "原始Ed25519公钥字符串",
      "rotation_proof": "旧私钥对新公钥哈希生成的Ed25519签名"
    }
    ```
2.  **存续性证明校验**：Anaphase 解析轮换事务，使用**旧公钥**校验轮换证明的签名有效性
3.  **原子切换生效**：校验通过后，Anaphase 内存中活跃公钥替换为新公钥，立即重载并使用新签名校验当前 `capability_mapping.toml`，同时记录该事件
4.  **密钥丢失恢复**：若私钥物理丢失，需完整修订 L0 `gene_lock.md` 文件；该操作会变更创世L0谱系哈希，强制系统在网络中执行全新身份注册

---

## 6. 错误码
当能力认证或校验发生失败时，执行层必须中止并返回以下诊断码：

| 错误码十进制 | 十六进制值 | 错误名称 | 说明 |
|:---|:---|:---|:---|
| `1408` | `0x0580` | `PERMISSION_DENIED` | 沙箱申请了其权限清单未声明的能力 |
| `1500` | `0x05DC` | `CONSENSUS_UNAVAILABLE` | 外部共识/人机校验引擎离线或不可达 |
| `1530` | `0x05FA` | `INVALID_CONFIGURATION_SIGNATURE` | 动态权限注册表文件签名校验失败 |
| `1531` | `0x05FB` | `UNAUTHORIZED_KEY_ROTATION` | 密钥轮换事务签名与当前活跃密钥校验不通过 |
| `1532` | `0x05FC` | `CONSENSUS_TIMEOUT` | 人机共识令牌超时，未收到有效签名响应 |
