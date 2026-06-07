> **最后核对日期**：2026-06-01 | **核对方式**：Context7 查官方文档
> **复核周期**：每季度（3/6/9/12 月）| **责任人**：维护者

# Spring AI / LLM 应用规范

> 适用场景：Java 后端集成 LLM（DeepSeek / 通义千问 / GLM），覆盖 Spring AI、LangChain4j。
> 与 backend-monolith.md / backend-microservice.md 配合使用。


## 一、技术栈

| 组件 | 版本 | 说明 | 不用 |
|------|------|------|------|
| **Spring AI** | 1.0.0+ | Spring 官方 AI 框架，统一 API | — |
| **LangChain4j** | 0.36+ | Java 版 LangChain，Tool Calling | — |
| **DeepSeek Java SDK** | 最新 | 国产主力，OpenAI 兼容 | — |
| ❌ OpenAI Java SDK | — | **禁止** | 国外 API |
| ❌ Anthropic Java SDK | — | **禁止** | 国外 API |


## 二、LLM Provider 选择（与 Python 规范对齐）

| 场景 | 推荐 | 备注 |
|------|------|------|
| **主力** | **DeepSeek** (`deepseek-chat` / `deepseek-reasoner`) | OpenAI 兼容协议、成本极低 |
| **备选 A** | 通义千问 (`qwen-max` / `qwen-plus`) | 阿里云百炼平台 |
| **备选 B** | 智谱 GLM (`glm-4-plus` / `glm-4-flash`) | 清华系 |
| **备选 C** | 零一万物 / 豆包 | 性价比高 |
| **本地部署** | vLLM / Ollama | 私有化部署 |

❌ **绝对禁止**：在业务代码中调用 OpenAI / Anthropic / Cohere 等任何国外 API。


## 三、Spring AI 集成模板

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    <!-- Spring AI 通过 OpenAI 兼容协议连接 DeepSeek -->
</dependency>
```

```yaml
# application.yml
spring:
  ai:
    openai:
      api-key: ${DEEPSEEK_API_KEY}
      base-url: https://api.deepseek.com
      chat:
        options:
          model: deepseek-chat
          temperature: 0.2
          max-tokens: 2048
```

```java
@RestController
@RequestMapping("/api/chat")
@RequiredArgsConstructor
public class ChatController {

    private final ChatModel chatModel;

    @PostMapping("/send")
    public R<String> chat(@RequestBody ChatRequest request) {
        // 第一步：构造 Prompt
        Prompt prompt = new Prompt(List.of(
            new SystemMessage("你是专业助手"),
            new UserMessage(request.getMessage())
        ));

        // 第二步：调用 LLM
        ChatResponse response = chatModel.call(prompt);
        String answer = response.getResult().getOutput().getContent();

        return R.ok(answer);
    }
}
```


## 四、Tool Calling 规范

```java
// 使用 Spring AI 的 Function Callback 机制
@Bean
public FunctionCallback weatherFunction() {
    return FunctionCallback.builder()
        .function("getWeather", (WeatherRequest req) -> {
            // 调用天气 API
            return new WeatherResponse(req.city(), "晴天", 25);
        })
        .description("获取指定城市的天气信息")
        .inputType(WeatherRequest.class)
        .build();
}
```


## 五、安全规范

| 项 | 规范 |
|------|------|
| **API Key** | 必须走 Nacos 配置中心或环境变量，**禁止**硬编码 |
| **Prompt 注入** | 用户输入消毒后再拼入 Prompt，**禁止**直接拼接 |
| **输出过滤** | LLM 输出必须经过内容审核后才返回前端 |
| **超时控制** | LLM 调用必须配置超时（默认 30s），防止阻塞 |
| **重试策略** | 只重试可恢复错误（5xx / timeout），**禁止**重试 4xx |
| **降级链** | 主力模型不可用 → 备选模型 → 本地模型 |


## 六、成本监控

- 每次 LLM 调用记录 token 使用量（input / output）
- 成本超过阈值自动告警
- 定期统计 Top 10 高成本用户


## 七、禁止事项

| 类别 | 禁止 |
|------|------|
| **模型合规** | ❌ 调用 OpenAI / Anthropic 等国外 API；❌ 引入 `openai-java` SDK |
| **安全** | ❌ API Key 硬编码；❌ 用户输入直接拼 Prompt；❌ LLM 输出不过滤 |
| **性能** | ❌ LLM 调用无超时；❌ 无重试/降级；❌ 在 Controller 层直接调 LLM |
| **架构** | ❌ Service 层直接调 LLM SDK（必须封装 Client）；❌ Prompt 硬编码在代码中 |

---

## 开发规则整合

### 架构设计
- 优先采用当前主流且经过生产验证的企业级方案
- 以中型公司实际落地标准设计
- 满足业务需求即可，不允许过度设计

### 编码原则
- 使用最少代码完成需求
- 优先可读性，其次是代码量
- 避免重复代码（DRY）

### 代码要求
- 所有代码必须包含中文注释
- 必须进行必要的判空处理
- 必须进行必要的异常处理

### 性能原则
- 先保证正确性
- 再保证可维护性
- 最后再考虑性能优化
