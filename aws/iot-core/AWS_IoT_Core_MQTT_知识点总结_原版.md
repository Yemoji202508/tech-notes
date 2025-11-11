# AWS IoT Core 与 MQTT 架构知识点总结

> 基于文章《从 HTTP 轮询到 MQTT：我们在 AWS IoT Core 上的架构演进与实战复盘》的学习笔记  
> 采用问答形式，聚焦易混淆和易遗忘的知识点

---

## 一、基础概念篇

### Q1: 为什么要从 HTTP 轮询迁移到 MQTT？

### HTTP 轮询的痛点
1. **电量焦虑**：频繁轮询快速消耗电池
2. **消息延迟与丢失**：服务端无法主动推送，发布期间消息丢失
3. **扩展性瓶颈**：有状态服务器难以水平扩展

### MQTT 的优势
- **轻量高效**：消息头部小，适合带宽受限环境
- **双向通信**：服务端和设备都可主动发送消息
- **高可扩展性**：Broker 独立扩展，后端服务灵活增减
- **解耦设计**：发布者和订阅者无需知道对方存在

---

## 二、MQTT 核心概念

### 基本架构
```
发布者（Publisher） → Broker → 订阅者（Subscriber）
                      ↓
                   Topic（主题）
```

### 核心组件
- **发布者（Publisher）**：消息发送方（如传感器设备）
- **订阅者（Subscriber）**：消息接收方（如后端服务）
- **主题（Topic）**：消息分类标签（如 `sensor/temperature/room1`）
- **Broker**：消息代理，负责路由转发

### Topic 的特性
- **动态性**：不需要预先创建，使用时自动存在
- **非持久化**：Topic 本身不占用存储空间
- **路由标识**：只在消息传递时作为路由标识
- **订阅关系持久化**：Broker 会维护"谁订阅了什么"的关系表

---

## 三、多设备场景下的 Topic 设计

### 推荐方案：每设备独立 Topic

**结构**：`{direction}/{deviceType}/{deviceId}/{action}`

**示例**：
- 设备 001 上行：`/dt/sensor/temp-001/report`
- 设备 002 上行：`/dt/sensor/temp-002/report`
- 设备 001 下行：`/cmd/sensor/temp-001/reboot`

### 服务端订阅策略：使用通配符

**不需要逐个订阅**，一次性订阅所有设备：
```
SUBSCRIBE /dt/sensor/#
```

### 通配符说明
- `+`（加号）：匹配单层级
  - `/dt/sensor/+/report` 匹配 `/dt/sensor/001/report`
- `#`（井号）：匹配多层级，必须在末尾
  - `/dt/sensor/#` 匹配 `/dt/sensor/001/report` 和 `/dt/sensor/001/temp/report`

---

## 四、Topic 设计最佳实践

### 1. 动静分离：上行下行要分明
- **上行数据**：`/dt/` 或 `/up/` 前缀（设备 → 云）
- **下行指令**：`/cmd/` 或 `/down/` 前缀（云 → 设备）

**好处**：权限控制精细化，设备只能发布上行、订阅下行

### 2. 层级清晰：包含关键实体信息
推荐结构：`{direction}/{deviceType}/{deviceId}/{action}`

**优势**：
- 路由更高效：通配符订阅无需解析消息体
- Rule 处理便捷：用 `topic(n)` 函数直接提取信息

### 3. 预留扩展空间
- AWS IoT Core 限制：最多 7 层，总长度 ≤ 256 字节
- 四层结构通常够用，还能预留扩展空间

### 4. 巧用通配符实现灵活订阅
- 订阅所有传感器：`/dt/sensor/#`
- 订阅特定类型：`/dt/sensor/temp-+/report`

---

## 五、MQTT vs RabbitMQ Topic

| 特性 | MQTT Topic | RabbitMQ Topic Exchange |
|------|-----------|------------------------|
| **定位** | 消息路由的地址标签 | Exchange + Routing Key 组合 |
| **持久化** | 不持久化，纯路由标识 | Queue 可持久化消息 |
| **消息存储** | 不存储，实时转发 | Queue 缓存消息，等待消费 |
| **离线消息** | 默认丢失（除非用 QoS） | Queue 保留，上线后继续消费 |
| **通配符** | `+`（单层）、`#`（多层） | `*`（单词）、`#`（多词） |
| **消息模型** | 发布/订阅（Pub/Sub） | 多种（Direct、Topic、Fanout） |
| **使用场景** | 物联网、实时推送、轻量通信 | 企业应用、任务队列、可靠性保证 |

---

## 六、连接与断开机制

### 长连接 vs 短连接

**MQTT 长连接**：
```
设备连接 → 保持连接 → 实时收发消息 → 主动断开
```

**HTTP 短连接（轮询）**：
```
连接 → 请求 → 断开 → 等待 → 连接 → 请求 → 断开...
```

### 消息处理流程

**订阅阶段**：
```
1. 订阅者连接 Broker
2. 执行 SUBSCRIBE /dt/sensor/001/report
3. Broker 在内存中记录订阅关系
```

**消息发布阶段**：
```
1. 发布者发送消息到 /dt/sensor/001/report
2. Broker 查询订阅关系表
3. 找到订阅者 → 转发消息
4. 消息处理完成 → 从 Broker 删除（不留痕迹）
5. 订阅关系和连接仍然保持
```

### 连接断开的触发条件

**主动断开**：
- 设备发送 DISCONNECT 消息
- Broker 主动断开（证书过期、违反策略）

**被动断开**：
- 网络故障（WiFi 断开、信号丢失）
- 设备断电/重启
- 设备进入休眠模式

**超时断开（Keep Alive 机制）**：
```
连接时设置 Keep Alive = 60 秒
设备每 60 秒发送 PINGREQ（心跳包）
Broker 回复 PINGRESP

如果 1.5 倍 Keep Alive 时间（90秒）内无消息
→ Broker 认为设备离线，断开连接
```

---

## 七、Broker 连接数与扩展性

### 自建 Broker 的限制
- 开源方案（Mosquitto、EMQX）：几千到几万连接
- 需要自己做集群、负载均衡、扩容

### AWS IoT Core 的优势
- **托管服务**：AWS 负责扩展性
- **海量连接**：支持百万级并发连接
- **无需关心**：底层服务器、集群、负载均衡
- **关注点**：不是性能瓶颈，而是成本（按连接数和消息数计费）

---

## 八、被动断开的补偿措施

### 1. Device Shadow（设备影子）- 核心方案
```
设备离线时：
- 应用程序修改 Shadow 状态
- Shadow 在云端缓存

设备重连后：
- 自动同步 Shadow 状态
- 设备执行相应操作
```

### 2. QoS（服务质量）等级
- **QoS 0**：最多一次，不保证送达
- **QoS 1**：至少一次，保证送达但可能重复
- **QoS 2**：恰好一次，保证送达且不重复

```python
# 重要消息使用 QoS 1 或 2
publish(topic, message, qos=1)
```

### 3. Persistent Session（持久会话）
```python
# 连接时设置 Clean Session = False
connect(clean_session=False)

# 设备断开后：
# - Broker 保留订阅关系
# - 缓存 QoS 1/2 消息
# - 重连后自动接收离线消息
```

### 4. Last Will（遗嘱消息）
```python
connect(
    lastWillTopic="/status/sensor/001",
    lastWillMessage="offline"
)

# 设备异常断开时：
# - Broker 自动发布遗嘱消息
# - 后端立即知道设备离线
```

### 5. 生命周期事件监控（AWS IoT Core）
```
订阅系统 Topic：
$aws/events/presence/connected/{clientId}
$aws/events/presence/disconnected/{clientId}

实时监控设备上下线
```

### 6. 应用层优化

**自动重连**：
```python
while True:
    try:
        client.connect()
        client.loop_forever()
    except:
        time.sleep(5)
        reconnect()
```

**消息队列缓冲**：
- 设备端：断网时本地缓存，重连后批量上传
- 服务端：使用 IoT Rule 转发到 SQS

**心跳优化**：
- 稳定网络：60-120 秒
- 不稳定网络：30 秒
- 省电设备：300 秒或按需连接

---

## 九、MQTT 与 HTTP 的适用场景

### MQTT 适合的场景
✅ 设备需要实时接收服务端推送  
✅ 设备频繁上报数据  
✅ 需要双向通信  
✅ 设备长期在线或需要快速响应  
✅ 对带宽和电量敏感  
✅ 大量设备的实时消息推送  

### HTTP 更适合的场景
✅ 请求-响应模式（查询、下单、登录）  
✅ RESTful API（CRUD 操作）  
✅ 复杂业务逻辑和状态码  
✅ 文件上传/下载  
✅ 浏览器访问（Web 应用）  
✅ 无状态的短连接场景  
✅ 需要缓存、代理、负载均衡  

### 混合架构（文章方案）
```
设备 ↔ Broker：MQTT（实时、双向、轻量）
Broker → 后端服务：HTTP（IoT Rule 转换）
用户 ↔ 后端：HTTP API
```

**实际项目组合**：
- 设备实时数据上报 → MQTT
- 设备接收指令 → MQTT
- 用户查询设备状态 → HTTP API
- 用户配置设备 → HTTP API
- 固件升级下载 → HTTP/HTTPS
- Web 管理后台 → HTTP

---

## 十、AWS IoT Core 核心组件

### 1. 设备网关（Device Gateway）
- 设备连接入口
- 支持 MQTT、WebSocket、HTTPS

### 2. 消息代理（Message Broker）
- 核心功能：MQTT Broker
- 安全、稳定、高性能

### 3. 身份验证和授权
- **设备端**：X.509 证书认证
- **服务端**：IAM Role（Access Key/Secret Key）
- **策略（Policy）**：JSON 文档定义权限

### 4. 设备影子（Device Shadow）
- 云端缓存设备状态副本
- 设备离线时可读写
- 重连后自动同步

### 5. IoT Rule（规则引擎）
监听 Topic，触发动作：
- 转发到 DynamoDB、Lambda、SQS
- 转发到另一个 Topic
- 转发到外部 HTTP 端点（文章采用）

**Rule 的价值**：
- 后端服务无需是 MQTT 客户端
- 降低改造和维护成本
- 专注业务逻辑

---

## 十一、架构设计要点

### 核心架构
```
上行路径（蓝色）：设备 → AWS IoT Core → 后端服务
下行路径（橙色）：后端服务 → AWS IoT Core → 设备
```

### 身份认证
- **设备端**：X.509 证书（每设备唯一）
- **服务端**：IAM Role
- **权限控制**：Policy 定义操作和 Topic 权限

### 消息上行
1. 设备连接到 AWS IoT Core 域名
2. 向预定义 Topic 发布消息
3. Message Broker 接收

### 消息下行
1. 后端通过 AWS SDK 发布消息
2. Message Broker 转发
3. 订阅该 Topic 的设备接收

---

## 十二、实战场景方案

### 高可靠场景（智能门锁）
- Device Shadow + QoS 1 + Persistent Session
- 遗嘱消息监控离线
- 自动重连

### 省电场景（传感器）
- 按需连接，不保持长连接
- 本地缓存 + 批量上传
- QoS 1 保证消息送达

### 实时监控场景
- 订阅生命周期事件
- 遗嘱消息 + 告警
- Device Shadow 查询最后状态

---

## 参考资料

1. [MQTT - The Standard for IoT Messaging](https://mqtt.org/)
2. [What is AWS IoT? - AWS IoT Core](https://docs.aws.amazon.com/iot/latest/developerguide/what-is-aws-iot.html)
3. [MQTT design best practices - AWS](https://docs.aws.amazon.com/whitepapers/latest/designing-mqtt-topics-aws-iot-core/designing-mqtt-topics-aws-iot-core.html)

---

**文档创建时间**：2025-11-11  
**基于文章**：《从 HTTP 轮询到 MQTT：我们在 AWS IoT Core 上的架构演进与实战复盘》
