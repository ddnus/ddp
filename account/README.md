# DDNUS 账号服务协议设计文档

## 1. 概述

本文档描述了 DDNUS 客户端与账号服务端之间的数据通信协议，基于 Protocol Buffers v3 定义。

### 1.1 协议文件

- **文件路径**: `ddp/account/account.v1.proto`
- **包名**: `account.v1`

### 1.2 设计目标

- **安全性**: 基于公私钥的身份验证，签名防篡改
- **一致性**: 版本号机制保证数据一致性
- **可扩展性**: 模块化设计，便于后续扩展
- **高效性**: 心跳机制实现按需同步

---

## 2. 服务接口定义

```protobuf
service AccountService {
    rpc Register (RegisterRequest) returns (RegisterResponse);    // 账号注册
    rpc Login (LoginRequest) returns (LoginResponse);             // 账号登录
    rpc Logout (LogoutRequest) returns (LogoutResponse);          // 账号登出
    rpc Update (UpdateRequest) returns (UpdateResponse);          // 更新账号信息
    rpc Heartbeat (HeartbeatRequest) returns (HeartbeatResponse); // 心跳保活
    rpc GetAccount (GetAccountRequest) returns (GetAccountResponse); // 获取账号信息
}
```

### 2.1 接口一览

| 接口 | 功能 | 认证要求 |
|------|------|----------|
| Register | 新用户注册账号 | 无需认证 |
| Login | 用户登录获取会话 | 公钥+签名 |
| Logout | 用户登出销毁会话 | session_token |
| Update | 更新账号信息 | session_token |
| Heartbeat | 心跳保活与数据同步 | session_token |
| GetAccount | 查询账号信息 | session_token |

---

## 3. 通用定义

### 3.1 错误码 (ErrorCode)

| 错误码 | 值 | 说明 |
|--------|-----|------|
| ERROR_CODE_UNSPECIFIED | 0 | 未指定 |
| ERROR_CODE_SUCCESS | 1 | 成功 |
| ERROR_CODE_INVALID_PARAM | 2 | 参数错误 |
| ERROR_CODE_UNAUTHORIZED | 3 | 未授权 |
| ERROR_CODE_NOT_FOUND | 4 | 未找到 |
| ERROR_CODE_ALREADY_EXISTS | 5 | 已存在 |
| ERROR_CODE_INTERNAL | 6 | 内部错误 |
| ERROR_CODE_RATE_LIMITED | 7 | 请求过于频繁 |
| ERROR_CODE_SIGNATURE_INVALID | 8 | 签名无效 |
| ERROR_CODE_NONCE_MISMATCH | 9 | Nonce不匹配 |
| ERROR_CODE_VERSION_CONFLICT | 10 | 版本冲突 |

### 3.2 请求元数据 (Metadata)

每个请求都应携带元数据，用于链路追踪和安全验证。

```protobuf
message Metadata {
    string request_id = 1;      // 请求ID，用于链路追踪
    int64 timestamp = 2;        // 请求时间戳（毫秒）
    string client_version = 3;  // 客户端版本
    string signature = 4;       // 请求签名（使用私钥签名）
}
```

### 3.3 响应状态 (ResponseStatus)

统一的响应状态结构。

```protobuf
message ResponseStatus {
    ErrorCode code = 1;  // 错误码
    string message = 2;  // 错误消息
}
```

---

## 4. 核心数据结构

### 4.1 账户信息 (Account)

```protobuf
message Account {
    int64 version = 1;               // 版本号，每次修改+1
    string user_id = 2;              // 用户ID
    string name = 3;                 // 账号名称
    string area = 4;                 // 所属区域
    string public_key = 5;           // 公钥
    int64 balance = 6;               // 余额（最小货币单位）
    int64 frozen = 7;                // 冻结金额
    int64 nonce = 8;                 // 交易次数
    AccountStatus status = 9;        // 账户状态
    int64 created_at = 10;           // 创建时间戳
    int64 updated_at = 11;           // 更新时间戳
    int64 last_login_at = 12;        // 最后登录时间戳

    repeated StorageEntry storages = 20; // 存储列表
    repeated AppEntry apps = 21;         // 应用列表
}
```

### 4.2 账户状态 (AccountStatus)

| 状态 | 值 | 说明 |
|------|-----|------|
| ACCOUNT_STATUS_UNSPECIFIED | 0 | 未指定 |
| ACCOUNT_STATUS_PENDING | 1 | 待激活 |
| ACCOUNT_STATUS_ACTIVE | 2 | 活跃 |
| ACCOUNT_STATUS_INACTIVE | 3 | 不活跃 |
| ACCOUNT_STATUS_SUSPENDED | 4 | 暂停 |
| ACCOUNT_STATUS_BANNED | 5 | 封禁 |

### 4.3 存储条目 (StorageEntry)

```protobuf
message StorageEntry {
    string storage_id = 1;    // 存储ID
    string name = 2;          // 存储名称
    StorageType type = 3;     // 存储类型 (SSD/HDD/NVMe)
    int64 used = 4;           // 已使用空间（字节）
    int64 total = 5;          // 总空间（字节）
    double price = 6;         // 每字节每秒价格（DDNUS）
    StorageStatus status = 7; // 存储状态
    int64 created_at = 8;     // 创建时间戳
    int64 updated_at = 9;     // 更新时间戳
}
```

### 4.4 应用条目 (AppEntry)

```protobuf
message AppEntry {
    string app_id = 1;                       // 应用ID
    string name = 2;                         // 应用名称
    string version = 3;                      // 应用版本
    int64 installed_at = 4;                  // 安装时间戳
    bool authorized = 5;                     // 是否已授权
    repeated PermissionEntry permissions = 6; // 权限列表
    string access_token = 7;                 // 访问令牌
    int64 token_expires_at = 8;              // 令牌过期时间
}
```

---

## 5. 接口详细设计

### 5.1 注册接口 (Register)

#### 请求 (RegisterRequest)

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| metadata | Metadata | 是 | 请求元数据 |
| username | string | 是 | 用户名 |
| public_key | string | 是 | 公钥 |
| area | string | 是 | 所属区域 |
| invite_code | string | 否 | 邀请码 |

#### 响应 (RegisterResponse)

| 字段 | 类型 | 说明 |
|------|------|------|
| status | ResponseStatus | 响应状态 |
| user_id | string | 分配的用户ID |
| account | Account | 账户信息 |

#### 流程图

```
┌─────────┐                              ┌─────────┐
│  客户端  │                              │  服务端  │
└────┬────┘                              └────┬────┘
     │                                        │
     │  RegisterRequest                       │
     │  (username, public_key, area)          │
     ├───────────────────────────────────────►│
     │                                        │
     │                                        │ 1. 验证参数
     │                                        │ 2. 检查用户名是否存在
     │                                        │ 3. 创建账户
     │                                        │ 4. 生成 user_id
     │                                        │
     │  RegisterResponse                      │
     │  (user_id, account)                    │
     │◄───────────────────────────────────────┤
     │                                        │
```

---

### 5.2 登录接口 (Login)

#### 请求 (LoginRequest)

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| metadata | Metadata | 是 | 请求元数据 |
| public_key | string | 是 | 公钥（用于标识用户） |
| timestamp | int64 | 是 | 登录时间戳 |
| signature | string | 是 | 使用私钥对 timestamp 签名 |
| device_id | string | 是 | 设备ID |
| device_info | string | 否 | 设备信息（JSON格式） |

#### 响应 (LoginResponse)

| 字段 | 类型 | 说明 |
|------|------|------|
| status | ResponseStatus | 响应状态 |
| session_token | string | 会话令牌 |
| session_expires_at | int64 | 会话过期时间戳 |
| account | Account | 账户信息 |
| server_time | int64 | 服务器时间戳 |

#### 身份验证流程

```
┌─────────┐                              ┌─────────┐
│  客户端  │                              │  服务端  │
└────┬────┘                              └────┬────┘
     │                                        │
     │  1. 生成 timestamp                     │
     │  2. 使用私钥签名 timestamp              │
     │                                        │
     │  LoginRequest                          │
     │  (public_key, timestamp, signature)    │
     ├───────────────────────────────────────►│
     │                                        │
     │                                        │ 1. 根据 public_key 查找用户
     │                                        │ 2. 使用 public_key 验证签名
     │                                        │ 3. 检查 timestamp 有效性
     │                                        │ 4. 生成 session_token
     │                                        │
     │  LoginResponse                         │
     │  (session_token, account)              │
     │◄───────────────────────────────────────┤
     │                                        │
```

#### 安全说明

- **签名验证**: 客户端使用私钥对 `timestamp` 签名，服务端使用存储的公钥验证
- **时间窗口**: 建议 `timestamp` 在服务器时间 ±5 分钟内有效，防止重放攻击
- **会话管理**: `session_token` 用于后续请求认证，有过期时间

---

### 5.3 登出接口 (Logout)

#### 请求 (LogoutRequest)

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| metadata | Metadata | 是 | 请求元数据 |
| session_token | string | 是 | 会话令牌 |
| logout_all_devices | bool | 否 | 是否登出所有设备 |

#### 响应 (LogoutResponse)

| 字段 | 类型 | 说明 |
|------|------|------|
| status | ResponseStatus | 响应状态 |

---

### 5.4 更新接口 (Update)

#### 请求 (UpdateRequest)

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| metadata | Metadata | 是 | 请求元数据 |
| session_token | string | 是 | 会话令牌 |
| expected_version | int64 | 是 | 期望的版本号（乐观锁） |
| nonce | int64 | 是 | 交易序号 |
| update_type | UpdateType | 是 | 更新类型 |
| new_name | string | 条件必填 | 新名称 |
| new_area | string | 条件必填 | 新区域 |
| new_public_key | string | 条件必填 | 新公钥 |
| old_key_signature | string | 条件必填 | 旧密钥签名（密钥轮换） |

#### 更新类型 (UpdateType)

| 类型 | 值 | 说明 |
|------|-----|------|
| UPDATE_TYPE_UNSPECIFIED | 0 | 未指定 |
| UPDATE_TYPE_NAME | 1 | 更新名称 |
| UPDATE_TYPE_AREA | 2 | 更新区域 |
| UPDATE_TYPE_PUBLIC_KEY | 3 | 更新公钥（密钥轮换） |

#### 响应 (UpdateResponse)

| 字段 | 类型 | 说明 |
|------|------|------|
| status | ResponseStatus | 响应状态 |
| account | Account | 更新后的账户信息 |

#### 乐观锁机制

```
┌─────────┐                              ┌─────────┐
│  客户端  │                              │  服务端  │
└────┬────┘                              └────┬────┘
     │                                        │
     │  UpdateRequest                         │
     │  (expected_version=5, nonce=10)        │
     ├───────────────────────────────────────►│
     │                                        │
     │                                        │ 1. 检查 session_token
     │                                        │ 2. 检查 expected_version == 当前版本
     │                                        │ 3. 检查 nonce == 账户nonce + 1
     │                                        │ 4. 执行更新
     │                                        │ 5. version++, nonce++
     │                                        │
     │  UpdateResponse (version=6)            │
     │◄───────────────────────────────────────┤
     │                                        │
```

**版本冲突处理**:
- 如果 `expected_version` 不等于当前版本，返回 `ERROR_CODE_VERSION_CONFLICT`
- 客户端需要重新获取最新数据后重试

---

### 5.5 心跳接口 (Heartbeat)

#### 请求 (HeartbeatRequest)

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| metadata | Metadata | 是 | 请求元数据 |
| session_token | string | 是 | 会话令牌 |
| client_version | int64 | 是 | 客户端本地账户数据版本 |
| last_sync_time | int64 | 否 | 上次同步时间戳 |

#### 响应 (HeartbeatResponse)

| 字段 | 类型 | 说明 |
|------|------|------|
| status | ResponseStatus | 响应状态 |
| server_time | int64 | 服务器时间戳 |
| server_version | int64 | 服务端账户数据版本 |
| need_sync | bool | 是否需要同步数据 |
| account | Account | 最新账户数据（need_sync=true时） |
| next_heartbeat_interval | int64 | 下次心跳间隔（毫秒） |

#### 数据同步流程

```
┌─────────┐                              ┌─────────┐
│  客户端  │                              │  服务端  │
└────┬────┘                              └────┬────┘
     │                                        │
     │  HeartbeatRequest                      │
     │  (client_version=5)                    │
     ├───────────────────────────────────────►│
     │                                        │
     │                                        │ 比较版本号
     │                                        │
     ├─────────────────────┬──────────────────┤
     │                     │                  │
     │  [版本一致]          │  [版本不一致]      │
     │                     │                  │
     │  HeartbeatResponse  │  HeartbeatResponse
     │  (need_sync=false)  │  (need_sync=true,
     │◄────────────────────┤   account=最新数据)
     │                     │◄─────────────────┤
     │                     │                  │
     │                     │  更新本地数据      │
     │                     │                  │
```

#### 心跳间隔策略

服务端可根据以下因素动态调整心跳间隔：
- 账户活跃度
- 服务器负载
- 网络状况

建议默认间隔：**30秒**

---

### 5.6 获取账号接口 (GetAccount)

#### 请求 (GetAccountRequest)

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| metadata | Metadata | 是 | 请求元数据 |
| session_token | string | 是 | 会话令牌 |
| target_user_id | string | 否 | 目标用户ID（查询他人） |
| target_public_key | string | 否 | 目标用户公钥（查询他人） |

#### 响应 (GetAccountResponse)

| 字段 | 类型 | 说明 |
|------|------|------|
| status | ResponseStatus | 响应状态 |
| account | Account | 账户信息 |

**注意**: 查询他人账户时，敏感信息（如余额）可能会被隐藏。

---

## 6. 安全设计

### 6.1 认证机制

```
┌────────────────────────────────────────────────────────────┐
│                       认证流程                              │
├────────────────────────────────────────────────────────────┤
│                                                            │
│   ┌──────────┐    公钥+签名     ┌──────────┐              │
│   │  客户端   │ ──────────────► │  服务端   │              │
│   │          │                  │          │              │
│   │ 持有私钥  │ ◄────────────── │ 存储公钥  │              │
│   └──────────┘   session_token  └──────────┘              │
│                                                            │
│   后续请求使用 session_token 进行认证                        │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 6.2 防重放攻击

- **Timestamp 验证**: 请求时间戳需在有效窗口内
- **Nonce 机制**: 每次交易 nonce 递增，防止重复提交

### 6.3 数据完整性

- **签名验证**: 关键操作需要签名验证
- **版本控制**: 乐观锁防止并发冲突

---

## 7. 错误处理

### 7.1 错误响应示例

```json
{
  "status": {
    "code": 2,
    "message": "参数错误: username 不能为空"
  }
}
```

### 7.2 客户端处理建议

| 错误码 | 处理方式 |
|--------|----------|
| SUCCESS | 正常处理 |
| INVALID_PARAM | 检查参数后重试 |
| UNAUTHORIZED | 重新登录 |
| NOT_FOUND | 提示用户 |
| ALREADY_EXISTS | 提示用户 |
| INTERNAL | 稍后重试 |
| RATE_LIMITED | 等待后重试 |
| SIGNATURE_INVALID | 检查密钥 |
| NONCE_MISMATCH | 刷新数据后重试 |
| VERSION_CONFLICT | 刷新数据后重试 |

---

## 8. 使用示例

### 8.1 完整登录流程

```
1. 客户端生成密钥对（首次使用）
   private_key, public_key = generate_keypair()

2. 注册账号
   RegisterRequest {
     metadata: { request_id: "uuid", timestamp: now() },
     username: "alice",
     public_key: public_key,
     area: "cn-east"
   }

3. 登录
   timestamp = now()
   signature = sign(private_key, timestamp)
   LoginRequest {
     metadata: { request_id: "uuid", timestamp: now() },
     public_key: public_key,
     timestamp: timestamp,
     signature: signature,
     device_id: "device-001"
   }

4. 获取 session_token，后续请求携带

5. 定期发送心跳
   HeartbeatRequest {
     metadata: { request_id: "uuid", timestamp: now() },
     session_token: session_token,
     client_version: local_account.version
   }
```

---

## 9. 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| v1.0 | 2024-12-13 | 初始版本 |

---

## 10. 附录

### 10.1 Proto 文件位置

```
ddp/
└── account/
    ├── account.v1.proto    # 协议定义文件
    └── README.md           # 本文档
```

### 10.2 相关链接

- [Protocol Buffers 文档](https://developers.google.com/protocol-buffers)
- [gRPC 文档](https://grpc.io/docs/)
