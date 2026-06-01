# ===== RocketMQ 消息队列规范 =====

> 适用于微服务架构中使用 RocketMQ 的模块，与 backend-microservice.md 配合使用

---

## 一、消息设计规范

### 1. Topic / Tag 命名

- **Topic 格式**：`{业务域}_{动作}`，如 `ORDER_CANCEL`、`PAY_NOTIFY`、`USER_REGISTER`
- **Tag 格式**：`{子类型}`，如 `TIMEOUT`、`USER_CANCEL`、`SYSTEM_CANCEL`
- 一个 Topic 对应一类业务事件，Tag 细分子类型
- 禁止所有消息都扔一个 Topic

### 2. 统一消息信封

```java
@Data
public class MqMessage<T> {
    /** 消息唯一 ID（用于链路追踪） */
    private String messageId;
    /** 业务单号（用于幂等去重） */
    private String bizId;
    /** 消息类型（对应 Tag） */
    private String messageType;
    /** 发送时间 */
    private LocalDateTime sendTime;
    /** traceId（链路追踪） */
    private String traceId;
    /** 业务数据体 */
    private T data;
}
```

### 3. 消息 Key 设置

- 每条消息**必须**设置 Key（便于控制台检索和问题排查）
- Key 格式：业务单号，如订单号、支付流水号

---

## 二、生产者规范

### 1. 发送方式选择

| 方式 | 适用场景 | 可靠性 | 吞吐量 |
|------|---------|--------|--------|
| **同步发送（syncSend）** | 订单创建、支付通知等重要消息 | 最高 | 低 |
| 异步发送（asyncSend） | 日志采集、数据同步等高吞吐场景 | 高 | 高 |
| 单向发送（sendOneWay） | 日志、监控打点等允许丢失的场景 | 低 | 最高 |
| **事务消息** | 分布式事务（下单+扣库存） | 最高 | 低 |

**默认用同步发送**，只在明确允许丢失时才用单向发送。

### 2. 生产者代码模板

```java
@Slf4j
@Component
public class OrderMqProducer {

    @Resource
    private RocketMQTemplate rocketMQTemplate;

    /**
     * 发送订单取消消息
     * @param dto 订单取消信息
     */
    public void sendOrderCancel(OrderCancelDTO dto) {
        MqMessage<OrderCancelDTO> message = new MqMessage<>();
        message.setMessageId(IdUtil.fastSimpleUUID());
        message.setBizId(dto.getOrderNo());
        message.setMessageType("ORDER_CANCEL_TIMEOUT");
        message.setSendTime(LocalDateTime.now());
        message.setTraceId(TraceIdUtil.get());  // 透传 traceId
        message.setData(dto);

        try {
            SendResult result = rocketMQTemplate.syncSend(
                "ORDER_CANCEL:TIMEOUT",
                MessageBuilder.withPayload(message)
                    .setHeader(MessageConst.PROPERTY_KEYS, dto.getOrderNo())
                    .build()
            );
            log.info("消息发送成功, orderId={}, msgId={}, sendStatus={}",
                dto.getOrderNo(), result.getMsgId(), result.getSendStatus());
        } catch (Exception e) {
            log.error("消息发送失败, orderId={}", dto.getOrderNo(), e);
            // 重要消息：落库后定时重试
            mqFailLogService.saveFailLog("ORDER_CANCEL", dto.getOrderNo(), message);
            throw new BizException("消息发送失败");
        }
    }
}
```

### 3. 发送失败兜底

重要消息发送失败后必须有兜底机制：
- 方案一：落库到 `mq_fail_log` 表，定时任务补偿发送
- 方案二：本地消息表 + 事务保证（发送消息和业务操作在同一事务中）

---

## 三、消费者规范

### 1. 消费模式

| 模式 | 适用场景 | 说明 |
|------|---------|------|
| **集群消费（CLUSTERING）** | 默认，绝大多数场景 | 同一条消息只被一个消费者处理 |
| 广播消费（BROADCASTING） | 缓存刷新、配置更新 | 所有消费者都收到 |

### 2. 消费者代码模板

```java
@Slf4j
@Component
@RocketMQMessageListener(
    topic = "ORDER_CANCEL",
    selectorExpression = "TIMEOUT",
    consumerGroup = "order-cancel-consumer-group",
    consumeMode = ConsumeMode.CONCURRENTLY
)
public class OrderCancelConsumer implements RocketMQListener<MqMessage<OrderCancelDTO>> {

    @Override
    public void onMessage(MqMessage<OrderCancelDTO> message) {
        log.info("收到消息, bizId={}, type={}, traceId={}",
            message.getBizId(), message.getMessageType(), message.getTraceId());

        // 1. 幂等校验（必须！）
        if (isProcessed(message.getBizId())) {
            log.info("消息已处理, 跳过, bizId={}", message.getBizId());
            return;
        }

        try {
            // 2. 业务处理
            orderService.cancelByTimeout(message.getData());

            // 3. 标记已处理
            markProcessed(message.getBizId());

            log.info("消息消费成功, bizId={}", message.getBizId());
        } catch (Exception e) {
            log.error("消息消费失败, bizId={}", message.getBizId(), e);
            throw e;  // 抛异常触发 MQ 自动重试
        }
    }

    private boolean isProcessed(String bizId) {
        String key = "mq:idempotent:ORDER_CANCEL:" + bizId;
        return Boolean.TRUE.equals(redisTemplate.hasKey(key));
    }

    private void markProcessed(String bizId) {
        String key = "mq:idempotent:ORDER_CANCEL:" + bizId;
        redisTemplate.opsForValue().set(key, "1", 24, TimeUnit.HOURS);
    }
}
```

### 3. Consumer Group 命名

- 格式：`{业务}-{功能}-consumer-group`
- 同一类消息的消费者必须用同一个 Consumer Group
- 不同业务消息禁止共用 Consumer Group

---

## 四、幂等消费

**MQ 保证至少投递一次，重复消费是常态，消费端必须做幂等。**

### 方案一：Redis SETNX（推荐）

```java
// key: mq:idempotent:{topic}:{bizId}, TTL: 24h
Boolean success = redisTemplate.opsForValue()
    .setIfAbsent("mq:idempotent:" + bizId, "1", 24, TimeUnit.HOURS);
if (!success) {
    return;  // 已处理过
}
```

### 方案二：数据库去重表

```sql
CREATE TABLE mq_message_dedup (
    id          BIGINT      NOT NULL AUTO_INCREMENT,
    biz_id      VARCHAR(64) NOT NULL COMMENT '业务单号',
    topic       VARCHAR(64) NOT NULL COMMENT '消息Topic',
    create_time DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uk_biz_topic (biz_id, topic)
) COMMENT '消息去重表';
```

---

## 五、死信队列（DLQ）

### 重试机制

- RocketMQ 默认重试 16 次，间隔递增：10s → 30s → 1min → 2min → ... → 2h
- 16 次重试后进入死信队列 `%DLQ%{consumerGroup}`

### 死信处理流程

```
消费失败 → 自动重试（16次） → 进入死信队列
                                    ↓
                           死信消费者（告警 + 落库）
                                    ↓
                           管理后台人工查看/重发
```

```java
@Slf4j
@Component
@RocketMQMessageListener(
    topic = "%DLQ%order-cancel-consumer-group",
    consumerGroup = "order-cancel-dlq-consumer-group"
)
public class OrderCancelDlqConsumer implements RocketMQListener<MessageExt> {

    @Override
    public void onMessage(MessageExt message) {
        String body = new String(message.getBody(), StandardCharsets.UTF_8);
        log.error("[死信消息] topic={}, tags={}, keys={}, body={}",
            message.getTopic(), message.getTags(), message.getKeys(), body);

        // 落库（供管理后台展示和人工重发）
        mqDeadLetterService.save(message.getTopic(), message.getKeys(), body);

        // 钉钉/邮件告警
        alertService.sendAlert("死信消息告警", "topic=" + message.getTopic() + ", keys=" + message.getKeys());
    }
}
```

---

## 六、顺序消息

**场景**：订单状态变更必须有序（创建 → 支付 → 发货 → 完成）

```java
// 生产者：同一订单号的消息发送到同一个 MessageQueue
rocketMQTemplate.syncSendOrderly(
    "ORDER_STATUS:ALL",
    MessageBuilder.withPayload(message).build(),
    orderId  // hashKey，相同 orderId 路由到同一队列
);

// 消费者：顺序消费模式
@RocketMQMessageListener(
    topic = "ORDER_STATUS",
    consumerGroup = "order-status-consumer-group",
    consumeMode = ConsumeMode.ORDERLY  // 顺序消费
)
public class OrderStatusConsumer implements RocketMQListener<MqMessage<OrderStatusDTO>> {
    // 同一 orderId 的消息会按顺序依次消费
}
```

---

## 七、事务消息

**场景**：下单 = 创建订单（DB）+ 通知库存服务（MQ），需要保证原子性

```java
// 1. 发送半消息 + 执行本地事务
@Component
public class OrderTransactionListener implements RocketMQLocalTransactionListener {

    @Override
    public RocketMQLocalTransactionState executeLocalMessage(Message msg, Object arg) {
        try {
            OrderDTO order = JSON.parseObject(new String(msg.getBody()), OrderDTO.class);
            orderService.createOrder(order);  // 本地事务：创建订单
            return RocketMQLocalTransactionState.COMMIT;  // 提交消息
        } catch (Exception e) {
            log.error("本地事务执行失败", e);
            return RocketMQLocalTransactionState.ROLLBACK;  // 回滚消息
        }
    }

    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        // MQ 回查：检查本地事务是否完成
        String orderId = msg.getHeaders().get("orderId", String.class);
        Order order = orderMapper.selectByOrderId(orderId);
        if (order != null) {
            return RocketMQLocalTransactionState.COMMIT;
        }
        return RocketMQLocalTransactionState.UNKNOWN;  // 继续回查
    }
}

// 2. 发送事务消息
rocketMQTemplate.sendMessageInTransaction(
    "ORDER_CREATE:NEW",
    MessageBuilder.withPayload(orderDTO).setHeader("orderId", orderId).build(),
    null  // arg 透传到 executeLocalMessage
);
```

---

## 八、消息追踪

```java
// 生产者：traceId 放入消息 Header
message.setTraceId(MDC.get("traceId"));
MessageBuilder.withPayload(message)
    .setHeader("traceId", MDC.get("traceId"))
    .build();

// 消费者：提取 traceId 放入 MDC
@Override
public void onMessage(MqMessage<T> message) {
    try {
        MDC.put("traceId", message.getTraceId());
        doBusiness(message);
    } finally {
        MDC.clear();
    }
}
```

---

## 九、禁止事项

- **禁止消息体过大**（> 4KB，大消息走数据库 + MQ 只通知 ID）
- **禁止消费端无幂等处理**（MQ 至少投递一次，重复是常态）
- **禁止消息丢失无兜底**（重要消息发送失败必须重试或落库）
- **禁止无死信队列监控**（死信消息必须有告警和人工处理流程）
- **禁止消费端做耗时操作**（消费超时会触发重试，耗时操作异步化）
- **禁止无 Topic 规划**（所有消息都扔一个 Topic，Tag 混乱）
- **禁止忽略消费失败日志**（必须记录完整的消息内容、bizId 和异常堆栈）
- **禁止消费者抛出异常不处理**（会导致无限重试，需要明确哪些异常需要重试、哪些直接跳过）
