BP:EMQX服务端，SpringBoot 客户端	Spring Integration MQTT （基于 Paho）	开发效率高，与 Spring 消息体系一致。
2. 使用设备唯一标识自动派生主题（推荐）
如果秤支持动态构建主题（部分工业网关支持），可利用其唯一标识符（如MAC地址、序列号）自动生成主题，无需逐一配置。
规则示例：在秤的固件中预设规则 scale/{MAC}/weight，秤启动时自动读取自己的MAC A4:CF:12:34:56:78，则最终发布主题为 scale/a4cf12345678/weight。
优点：随插即用，无需任何手工配置。客户端只需知道设备MAC或扫描二维码即可构建主题进行订阅。
缺点：要求设备固件支持动态替换变量。
install on ubuntu
 sudo apt-add-repository ppa:mosquitto-dev/mosquitto-ppa
 1305  sudo apt install mosquitto mosquitto-clients
 1306  systemctl status mosquitto
 1307  history

检查端口监听
bash
ss -lntp | grep 1883
# 或
netstat -lntp | grep mosquitto

MQTT（Message Queuing Telemetry Transport）是一种基于发布/订阅模型的轻量级物联网消息协议，特别适用于低带宽、高延迟或网络不稳定的场景。以下将从核心概念到生产最佳实践，系统性地梳理其使用方案。

---

### 1. 选型场景
MQTT适合以下场景，不适用时需谨慎：
- **物联设备通信**：传感器、执行器、移动端实时数据上报。
- **弱网环境**：卫星通信、偏远地区工业设备、车联网。
- **高并发长连接**：百万级同时在线的Topic发布/订阅。
- **不适合**：大文件传输（协议本身轻量，传输大数据需分片或旁路）、强事务性系统（如金融交易），这类场景可考虑AMQP。

---

### 2. 核心概念与QoS选择

| QoS值 | 描述 | 使用场景举例 | 注意事项 |
|------|------|------------|---------|
| **0** | 最多一次（丢失不重发） | 高频传感器数据，丢失一帧影响不大 | 网络稍有不稳即可能丢消息 |
| **1** | 至少一次（可能重复） | 设备状态上报、告警 | 接收端需幂等处理 |
| **2** | 精确一次（开销最大） | 控制指令、计费信息 | 增加延迟，谨慎使用 |

**最佳实践**：
- 永远避免在发布端使用QoS 2，除非接收端有严格的去重逻辑且资源充足。
- 对多数实时数据采用QoS 1，业务层做幂等；关键指令可降级为QoS 1 + 应答报文。
- 根据网络质量动态协商QoS（客户端库通常支持降级配置）。

---

### 3. 主题（Topic）设计规范

一个好的主题设计能极大降低系统复杂度：

**结构层次化**：
```
{公司}/{部门}/{设备类型}/{设备ID}/{数据类别}
例如：acme/workshop01/temp_sensor/TX1000/status
```

**命名约定**：
- 使用小写字母、数字和下划线。
- 避免开头/结尾的斜杠。
- 禁止空格和特殊字符。
- 使用通配符时谨慎：
  - `+` 匹配单层：`acme/+/temp_sensor`
  - `#` 匹配多层，**必须放在主题末尾**，且订阅时避免过度拉取数据。

**反模式**：
- 不要用 `#` 订阅根级所有主题，会导致Broker负载激增。
- 不要把设备唯一标识作为主题的顶层，应分层便于管理和权限控制。

---

### 4. 安全防护

#### 传输加密
- 生产环境**必须启用TLS/SSL**（端口8883）。
- 使用强密码套件，禁用低版本TLS（≥1.2）。
- 嵌入式设备推荐使用预共享密钥（PSK）减少证书管理开销。

#### 认证方式
- **用户名/密码**：适合简单的内部系统，但密码需加密存储。
- **X.509证书**：大规模部署时推荐，设备固件内置独立证书，可实现双向认证（服务端验证客户端，客户端也验证服务端）。
- **JWT令牌**：适合Web/App集成，较短生命周期。

#### 授权（ACL）
- 基于角色的细粒度权限控制，例如：
  - 传感器只能 `publish` 到 `device/{id}/data`
  - 控制器只能 `subscribe` `cmd/{controller_id}/#`
- 使用Broker内置ACL（如EMQX的规则引擎或Mosquitto的ACL文件）。

---

### 5. 连接管理最佳实践

- **Client ID唯一性**：每个连接必须唯一，建议格式 `{设备类型}-{序列号}-{进程ID}`，例如 `temp-TX1000-1234`。
- **Keep Alive**：
  - 根据网络质量设置心跳间隔，一般为60-300秒。
  - 弱网环境下适当缩短，但不要低于10秒以避免频繁断连。
- **自动重连**：
  - 客户端实现指数退避重连：初始1秒，每次翻倍，最大60秒。
  - 重连后重新订阅主题（若Broker不支持持久会话需手动恢复）。
- **遗嘱消息（LWT）**：
  - 每个设备连接时设置遗嘱，当异常断线时Broker发布离线消息，其他设备可感知状态变化。
  - 遗嘱主题应与设备状态主题分离，如 `device/{id}/status` 通常由设备手动发布，遗嘱则发布到 `device/{id}/will`。

---

### 6. 离线消息与持久会话

- **Clean Session = false**：
  - 设置`cleanStart = false`（MQTT 5.0）或`Clean Session = 0`（MQTT 3.1.1）开启持久会话，Broker会为客户端缓存离线期间的消息。
  - 订阅时，使用`Session Expiry Interval`控制会话保留时间（MQTT 5.0）。
  - 对偶然断网的设备非常有用，但需注意Broker内存/磁盘消耗，建议设置合理的会话过期时间。

- **保留消息（Retained）**：
  - 对每个主题发布`retained`消息，新订阅者立即可获得最新值，适合配置类信息。
  - 避免滥用，定期清理无用的保留消息，否则Broker内存会不断增长。

---

### 7. 高性能与水平扩展

- **Broker集群**：
  - 使用支持分布式的Broker（如EMQX、HiveMQ、VerneMQ），通过集群节点水平扩展连接和吞吐。
  - 典型社区版EMQX支持百万级并发，通过`emqx_ctl cluster join`组建集群。

- **负载均衡**：
  - 在Broker前端部署TCP负载均衡器（如HAProxy、Nginx stream），基于客户端源IP哈希保持会话粘性。
  - 避免客户端直接连接到后端Broker节点，方便弹性扩缩。

- **流量控制**：
  - 限制每个客户端订阅的主题数量（如EMQX的`zone.external.max_subscriptions`）。
  - 对高频发布主题设置消息速率配额，防止个别设备风暴影响整体。

---

### 8. 数据持久化与集成

MQTT只是传输层，需要将数据沉淀到后端系统：

- **规则引擎**：Broker内置（如EMQX Rule Engine、HiveMQ Extension）可将消息流转至Kafka、MySQL、InfluxDB、HTTP服务等，无需额外编程。
- **消息桥接**：通过`mqtt-bridge`将消息从边缘Broker同步到云端中心Broker。
- **旁路持久化**：使用独立的订阅者消费所有消息并写入时序数据库（如TDengine、TimescaleDB），不影响设备连接。

---

### 9. 监控与运维

- **关键指标**：
  - 连接数、消息发送/接收速率、QoS分布、字节流量。
  - 会话数量、保留消息数。
- **Broker内置监控**：
  - EMQX提供了丰富的Prometheus指标，配合Grafana面板。
  - Mosquitto可通过`$SYS`主题发布系统状态。
- **日志级别**：生产环境设置`warning`或`error`，避免磁盘写满。

---

### 10. 客户端库选择

根据开发语言和平台选择合适的MQTT库，优先选择支持MQTT 5.0的版本：

| 语言 | 推荐库 |
|------|--------|
| Python | paho-mqtt, gmqtt（异步） |
| JavaScript/Node.js | mqtt.js, eclipse-mosquitto |
| Java | Eclipse Paho Java, HiveMQ MQTT Client |
| C/C++ | Eclipse Paho C, Mosquitto C library |
| Android/iOS | Eclipse Paho Android/MQTT-SN |
| 嵌入式 | wolfMQTT, coreMQTT (FreeRTOS) |

---

### 总结：生产环境部署清单

1. 启用TLS双向认证，配置ACL。
2. 主题命名按业务层次规划，避免滥用通配符。
3. 设备端设置合理的Keeplive和重连策略，携带遗嘱消息。
4. Broker集群部署，前端加负载均衡。
5. 使用规则引擎或独立消费者将关键数据持久化。
6. 监控连接数、消息速率，设置资源配额。
7. 定期清理过期的保留消息和无用会话。
8. 根据数据重要性选择QoS，控制指令考虑QoS 1 + 业务确认。

遵循以上实践，MQTT可稳定承载从几十到数百万设备的物联网业务。
