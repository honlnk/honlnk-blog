# Spring AI Alibaba 动态配置API Key和模型指南

> 本文档说明如何在Spring AI Alibaba框架中通过编程方式配置API Key和模型，而无需在配置文件中强制配置。

## 1. 概述

Spring AI Alibaba是一个基于Spring Boot的AI应用开发框架，它允许开发者通过编程方式创建和配置模型实例，而不强制要求在配置文件中设置API Key和模型参数。

## 2. 核心概念

Spring AI Alibaba支持通过编程方式创建模型实例，这意味着API Key和模型配置可以来自：
- 用户表单输入
- 数据库配置
- API接口返回
- 环境变量
- 其他任何动态来源

## 3. 实现方法

### 3.1 编程方式创建DashScope模型

```java
import com.alibaba.cloud.ai.dashscope.api.DashScopeApi;
import com.alibaba.cloud.ai.dashscope.chat.DashScopeChatModel;
import org.springframework.ai.chat.model.ChatModel;

// 从动态来源获取API Key（示例：用户输入）
String apiKey = getUserApiKey(); // 从用户输入、数据库等来源获取

// 创建DashScope API实例
DashScopeApi dashScopeApi = DashScopeApi.builder()
    .apiKey(apiKey)
    .build();

// 创建ChatModel实例
ChatModel chatModel = DashScopeChatModel.builder()
    .dashScopeApi(dashScopeApi)
    .build();
```

### 3.2 在Agent中使用动态模型

```java
import com.alibaba.cloud.ai.graph.agent.ReactAgent;

ReactAgent writerAgent = ReactAgent.builder()
    .name("writer_agent")
    .model(chatModel)  // 使用动态创建的模型实例
    .description("可以写文章。")
    .instruction("你是一个知名的作家，擅长写作和创作。请根据用户的提问进行回答。")
    .build();
```

### 3.3 在Spring Boot应用中的完整示例

```java
@RestController
public class ChatController {
    
    // 从用户输入创建动态Agent
    @PostMapping("/create-agent")
    public ResponseEntity<String> createAgent(@RequestBody AgentConfig config) {
        // 从请求中获取API Key和其他配置
        DashScopeApi dashScopeApi = DashScopeApi.builder()
            .apiKey(config.getApiKey())  // 来自用户输入
            .build();
            
        ChatModel chatModel = DashScopeChatModel.builder()
            .dashScopeApi(dashScopeApi)
            .build();
        
        ReactAgent agent = ReactAgent.builder()
            .name("dynamic_agent")
            .model(chatModel)  // 使用动态创建的模型
            .instruction(config.getInstruction())
            .build();
        
        // 调用Agent
        AssistantMessage response = agent.call(config.getUserInput());
        return ResponseEntity.ok(response.getText());
    }
}

// 配置数据类
class AgentConfig {
    private String apiKey;
    private String instruction;
    private String userInput;
    
    // getter和setter方法
    public String getApiKey() { return apiKey; }
    public void setApiKey(String apiKey) { this.apiKey = apiKey; }
    public String getInstruction() { return instruction; }
    public void setInstruction(String instruction) { this.instruction = instruction; }
    public String getUserInput() { return userInput; }
    public void setUserInput(String userInput) { this.userInput = userInput; }
}
```

## 4. 动态配置的优势

1. **灵活性**：API Key和模型配置可以从任意来源获取
2. **安全性**：避免将敏感信息硬编码在配置文件中
3. **多租户支持**：可以为不同用户使用不同的API Key
4. **动态切换**：支持在运行时切换模型提供商

## 5. 其他模型提供商

Spring AI Alibaba同样支持其他模型提供商（如OpenAI、DeepSeek等）的动态配置，它们都有类似的编程创建方式。

## 6. 注意事项

1. 确保API Key的安全性，避免在日志或响应中暴露
2. 考虑API Key的验证和错误处理
3. 根据需要实现缓存机制以提高性能
4. 考虑连接池和资源管理

## 7. 结论

Spring AI Alibaba框架设计上完全支持动态配置API Key和模型参数，不强制要求在配置文件中设置，开发者可以从任何来源获取配置信息并在运行时创建模型实例。