---
layout:     post
title:     DeepSeek+langchain4j的使用
subtitle:   通过langchain4j接入deepSeek
date:       2025-02-05
author:     catmai
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Java
    - langchain4j
    - AI
    - DeepSeek
---


​		各位读者新年好呀，deepSeek在春节假期着实是火了一把，低成本和高性能让对岸的许多公司破防了。那么既然有这么好用的LLM，我们Javaer应该如何来使用呢？

# langchain4j

​		读过我的AI篇的读者应该还记得，java有自己的langchain，而langchain4j在24年底推出了第一个大版本！（1.0.0-alpha1）/撒花

​		查看deepseek的API文档，可以看到deepseek是适配openAi的gpt接口的，那么事情变得简单了起来。我们可以直接使用langchain4j的openAi包来解决使用deepseek。

![](https://raw.githubusercontent.com/CatMaiV/BlogImage/main/img/20250205113350.png)

​		不过deepseek这里缺少了embedding的接口，不知道未来是不是会推出相关的接口。



# 依赖

第一步还是先引入依赖，我们需要如下内容：

``` xml
<!--langchain4j配置-->
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-spring-boot-starter</artifactId>
    <version>${langchain4j.version}</version>
</dependency>

<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
    <version>${langchain4j.version}</version>
</dependency>
```

创建一个deepseek的配置类，方便维护deepseek的apikey

``` java
@Component
@ConfigurationProperties(prefix = "langchain.deepSeek")
public class DeepSeekConfig {

    private static String apiKey;

    private static String baseUrl;

    private static String modelName;

    private static Integer maxTokens;


    public static String getApiKey() {
        return apiKey;
    }

    public void setApiKey(String apiKey) {
        DeepSeekConfig.apiKey = apiKey;
    }

    public static String getBaseUrl() {
        return baseUrl;
    }

    public void setBaseUrl(String baseUrl) {
        DeepSeekConfig.baseUrl = baseUrl;
    }

    public static String getModelName() {
        return modelName;
    }

    public void setModelName(String modelName) {
        DeepSeekConfig.modelName = modelName;
    }

    public static Integer getMaxTokens() {
        return maxTokens;
    }

    public void setMaxTokens(Integer maxTokens) {
        DeepSeekConfig.maxTokens = maxTokens;
    }

}
```

配置文件如下：

``` yaml
# langchain4j配置
langchain:
    deepseek:
        apiKey: xxxxx
        baseUrl: https://api.deepseek.com
        modelName: deepseek-chat
        maxTokens: 
```



# 低阶用法

​		langchain4j的低阶用法是创建一个ChatLanguageModel对象，这个对象封装好了常见的chat接口，我们可以使用这个类就像使用LLM的SDK一样，一些接口中的参数也都封装好了，使用起来非常方便。可以看一下ChatLanguageModel的源码，dev.langchain4j.model.chat包下。

我们使用OpenAiChatModel来实例化，简单的写法如下：

``` java
ChatLanguageModel model = OpenAiChatModel.builder()
                .apiKey(DeepSeekConfig.getApiKey())
                .baseUrl(DeepSeekConfig.getBaseUrl())
                .maxTokens(DeepSeekConfig.getMaxTokens())
                .modelName(StringUtils.isEmpty(modelName) ? DeepSeekConfig.getModelName() : modelName)
                .logRequests(true)
                .logResponses(true)
                .build();
```

还有一些其他额外的配置，阅读OpenAiChatModel的源码可以看到有这些初始化内容。

![](https://raw.githubusercontent.com/CatMaiV/BlogImage/main/img/20250205114559.png)

![](https://raw.githubusercontent.com/CatMaiV/BlogImage/main/img/20250205114747.png)

由于deepseek完全兼容OpenAI的API，所以我们这里只是替换了OpenAiChatModel的baseUrl，就可以把服务切换到deepseek，非常的方便。



## 不兼容的地方

虽然API完全兼容，但是OpenAI的API依然多了一些更灵活的参数，例如strictJsonSchema。我们可以查阅deepseek官方的对话接口，查看可以允许有哪些参数，从而知晓OpenAiChatModel中会生效的配置是哪些。



![](https://raw.githubusercontent.com/CatMaiV/BlogImage/main/img/20250205115432.png)



## 对话

查看ChatLanguageModel的源码可以知道，主要有一下几个方法可以用

```java
default String generate(String userMessage);


default Response<AiMessage> generate(ChatMessage... messages);


Response<AiMessage> generate(List<ChatMessage> messages);


default Response<AiMessage> generate(List<ChatMessage> messages, List<ToolSpecification> toolSpecifications);


default Response<AiMessage> generate(List<ChatMessage> messages, ToolSpecification toolSpecification);
```

其中最简单的就是入参一个字符串，userMessage，返回值也是字符串，是LLM的回复，下面我们来看下其他的对象。



## ChatMessage

![](https://raw.githubusercontent.com/CatMaiV/BlogImage/main/img/20250205121842.png)

ChatMessage是一个接口，共有四种实现，分别对应了MessageList的四种消息类型。



### AiMessage

Ai回复包含了文本内容和tool调用请求对象，如果Ai回复内存在Tool请求，那么langchain还会处理tool请求。

```java
public class AiMessage implements ChatMessage {

    private final String text;
    private final List<ToolExecutionRequest> toolExecutionRequests;
 
    ...
}
```



### SystemMessage

系统消息，这个部分是系统提示词部分，只有一个text，可以作为LLM的基础设定。

```java
public class SystemMessage implements ChatMessage {

    private final String text;
    
    ...
}
```



### UserMessage

用户消息，包含名称和内容。内容可以传递多模态的内容，例如图片、视频等，需要LLM支持多模态内容解析。

```java
public class UserMessage implements ChatMessage {

    private final String name;
    private final List<Content> contents;
    
    ...
}
```



### ToolExecutionResultMessage

工具执行结果消息

```java
public class ToolExecutionResultMessage implements ChatMessage {

    private final String id;
    private final String toolName;
    private final String text;
 
    ...
}
```



### 完整示例

```java
ChatLanguageModel deepSeek = OpenAiChatModel.builder()
                .apiKey(DeepSeekConfig.getApiKey())
                .baseUrl(DeepSeekConfig.getBaseUrl())
                .maxTokens(DeepSeekConfig.getMaxTokens())
                .modelName(StringUtils.isEmpty(modelName) ? DeepSeekConfig.getModelName() : modelName)
                .logRequests(true)
                .logResponses(true)
                .build();
List<ChatMessage> chatMessages = new ArrayList<>();

List<Content> contents = new ArrayList<>();
contents.add(TextContent.from("用户消息"));
contents.add(ImageContent.from("图片URL"));

UserMessage userMessage = UserMessage.from("用户1",contents);
SystemMessage systemMessage = SystemMessage.from("你是一个AI助手");

chatMessages.add(systemMessage);
chatMessages.add(userMessage);

Response<AiMessage> response = deepSeek.generate(chatMessages);
```





# 高阶用法

高阶用法是使用注解和AiService来达到使用的目的，并且可以更方便的配置memeryStore（历史记录存储）、toolSet（工具集）、retriever（内容取回）等。



如果需要使用，我们需要定义一个AiService

```java
@AiService
public interface Assistant {


    @SystemMessage({
            "{{prompt}}",
            "今天是 {{current_date}}."
    })
    Response<AiMessage> chat(@MemoryId String memoryId, @V("prompt") String prompt, @UserMessage String message);

}
```



@SystemMessage 是系统提示词注解，可以通过@V来传递入参，在入参内通过@UserMessage注解来声明用户消息。



## 完整示例

``` java
ChatLanguageModel deepSeek = OpenAiChatModel.builder()
                .apiKey(DeepSeekConfig.getApiKey())
                .baseUrl(DeepSeekConfig.getBaseUrl())
                .maxTokens(DeepSeekConfig.getMaxTokens())
                .modelName(StringUtils.isEmpty(modelName) ? DeepSeekConfig.getModelName() : modelName)
                .logRequests(true)
                .logResponses(true)
                .build();

Assistant assistant = AiServices.create(Assistant.class, deepSeek);

//对话
Response<AiMessage> answer = assistant.chat("memeryId","你是一个AI助手","你好呀");
```

这里我们同样需要创建ChatLanguageModel，因为高阶用法是基于低阶的，以上是最简单的示例。

我们可以使用AiServices.builder来构建，可以阅读源码查看有哪些可配置选项。

![](https://raw.githubusercontent.com/CatMaiV/BlogImage/main/img/20250205124203.png)

可以看到，高阶用法非常简单，并且帮助用户完成了retrievalAugmentor、toolSpecifications等，我们可以只关心功能本身而不用关注LLM的接口实现和内容取回的过程。



## 开启日志

如果需要在控制台中查看调用LLM的日志，需要在yml中调整logging等级。

``` yaml
logging:
    level:
      dev.langchain4j: debug
      dev.ai4j.openai4j: debug
```



# 总结

​		本篇介绍了如何使用langchain4j接入deepSeek，使得deepSeek应用开发难度大大下降，并且langchain4j还提供了额外的功能，后续会分享如何取回内容以及自定义tool。







