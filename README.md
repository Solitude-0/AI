# AI Week 1 Starter：Java 21 + Spring Boot + Spring AI

一个用于 AI 大模型学习第一周的最小可扩展工程，包含同步对话、结构化输出、SSE 流式响应、参数校验、统一异常处理和单元测试。

## 1. 技术版本

- Java 21
- Spring Boot 4.1.1（稳定版）
- Spring AI 2.0.1（稳定版）
- Spring MVC 提供 REST API；引入 `spring-webflux` 仅用于 `ServerSentEvent`/Reactor 流式类型，不使用 WebFlux Starter 替换 MVC。
- Maven
- OpenAI Chat API（默认模型 `gpt-5-mini`，可通过环境变量覆盖）

Spring AI 2.0.x 官方支持 Spring Boot 4.0.x / 4.1.x。请确认你使用的模型服务提供方支持配置中的模型名称；如使用 OpenAI 兼容 API，可在后续增加 `spring.ai.openai.base-url` 配置。

## 2. 目录结构

```text
ai-week1-starter/
├── pom.xml
├── README.md
└── src
    ├── main
    │   ├── java/com/example/aiweek1
    │   │   ├── AiWeek1Application.java
    │   │   ├── ai
    │   │   │   ├── ChatGateway.java
    │   │   │   └── SpringAiChatGateway.java
    │   │   ├── api
    │   │   │   ├── AiController.java
    │   │   │   ├── dto
    │   │   │   └── exception
    │   │   └── service/AiService.java
    │   └── resources/application.yml
    └── test/java/com/example/aiweek1/service/AiServiceTest.java
```

## 3. 环境准备

1. 安装 JDK 21。
2. 安装 Maven 3.9+。
3. 配置一个有权限调用所选模型的 API Key。
4. 确保本机能访问 Maven Central 和你配置的模型 API 服务。

检查版本：

```bash
java -version
mvn -version
```

## 4. 配置 API Key

### macOS / Linux

```bash
export OPENAI_API_KEY='替换为自己的 API Key'
export OPENAI_MODEL='gpt-5-mini'
```

### Windows PowerShell

```powershell
$env:OPENAI_API_KEY = '替换为自己的 API Key'
$env:OPENAI_MODEL = 'gpt-5-mini'
```

不要把真实 API Key 写入 `application.yml`、提交到 Git 或贴进问题截图。
如果使用 OpenAI 兼容服务，通常还需要在 `application.yml` 中配置其文档规定的 Base URL；不同兼容服务的模型名、结构化输出支持程度可能不同。

## 5. 运行测试

在项目根目录执行：

```bash
mvn test
```

当前测试是纯单元测试，通过 Mock `ChatGateway` 测试业务校验与异常转换，不调用真实模型，因此运行测试不需要设置 `OPENAI_API_KEY`。

## 6. 启动应用

先按第 4 节设置好 API Key，然后执行：

```bash
mvn spring-boot:run
```

或先打包再运行：

```bash
mvn clean package
java -jar target/ai-week1-starter-0.0.1-SNAPSHOT.jar
```

服务默认监听 `http://localhost:8080`。

## 7. 接口验证

### 7.1 普通对话

```bash
curl -sS -X POST 'http://localhost:8080/api/ai/chat' \
  -H 'Content-Type: application/json' \
  -d '{"message":"用三句话解释什么是大语言模型"}'
```

成功响应示例（模型输出会变化）：

```json
{
  "content": "……",
  "model": "gpt-5-mini",
  "createdAt": "2026-10-20T08:00:00Z"
}
```

### 7.2 结构化输出

```bash
curl -sS -X POST 'http://localhost:8080/api/ai/analyze' \
  -H 'Content-Type: application/json' \
  -d '{"text":"我们发布了一个新版本，修复了多个缺陷，并提升了系统稳定性。"}'
```

返回字段包含 `title`、`summary`、`keywords` 与 `sentiment`。`sentiment` 必须是 `POSITIVE`、`NEUTRAL` 或 `NEGATIVE`。
注意：结构化输出可约束格式，但不保证事实绝对正确；服务端会再次执行 Bean Validation。

### 7.3 SSE 流式输出

```bash
curl -N -G 'http://localhost:8080/api/ai/stream' \
  --data-urlencode 'message=解释一下 Java 中的虚拟线程' \
  -H 'Accept: text/event-stream'
```

事件格式示例：

```text
event:token
data:{"content":"虚拟线程","code":null,"message":null}

event:done
data:{"content":null,"code":null,"message":null}
```

- `token`：一个文本片段，不保证每片恰好对应一个词。
- `error`：流已经开始后发生异常时发送，包含稳定错误码和可展示的错误信息。
- `done`：流完成标记。

### 7.4 验证统一参数错误

```bash
curl -i -X POST 'http://localhost:8080/api/ai/chat' \
  -H 'Content-Type: application/json' \
  -d '{"message":"   "}'
```

应返回 HTTP 400，并包含 `VALIDATION_ERROR` 或 `INVALID_REQUEST` 错误码。

## 8. 超时、重试与安全说明

- `spring.ai.openai.timeout: 30s`：限制模型 HTTP 请求时间。
- `spring.mvc.async.request-timeout: 60s`：限制 MVC 异步请求时长，应大于模型调用超时。
- `spring.ai.openai.max-retries: 1` 与 `spring.ai.retry.max-attempts: 1`：设置较保守的重试行为，避免默认重试叠加导致延迟或费用放大。
- 模型调用错误只返回通用信息，不把异常堆栈、凭据或供应方原始响应返回给客户端。
- SSE 响应开始后 HTTP 状态码和响应头已提交，后续异常通过 `error` 事件通知，而不是尝试再返回普通 JSON。
- 生产环境还需要鉴权、限流、审计、内容脱敏、成本配额、请求取消和调用链监控；本项目未包含用户认证。

## 9. 学习任务

1. 给 `ChatRequest` 增加会话 ID，并讨论多轮上下文如何存储。
2. 为普通对话记录 Token 用量、延迟和请求 ID。
3. 给模型失败增加可控重试策略，并证明重试不会被重复叠加。
4. 为结构化输出建立一组固定输入样例，统计结构化解析成功率。
5. 比较 SSE 与普通 JSON 接口在首 token 延迟、连接中断和错误处理上的差异。

## 10. 兼容性备注

这是按上述官方稳定版本编写的示例项目。当前执行环境未安装 Maven 且无法访问 Maven Central，因此无法在生成环境中实际执行 `mvn test` 或验证依赖解析；请在有网络的本地开发环境运行上述命令。如果构建报错，请优先核对 Maven Central 可用性、JDK/Maven 版本、模型服务端点和所选模型是否可用。
