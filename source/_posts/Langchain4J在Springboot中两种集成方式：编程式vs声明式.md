---
title: Langchain4J在Springboot中两种集成方式：编程式vs声明式
date: 2026-05-03
tags:
  - Langchain4J
---

## 前言

最近在学习Langchain4J，考虑通过Langchain4J调用Agent用于数据安全策略的自动化更新与迭代，但是我之前没有接触过Java，因此考虑速成使用，在学习Langchan4J的时候，我了解到两种不同的使用方法，因此作为一个记录和对比。  
在传统的`Spring`开发中，我们通常会写一个 `ConsultantServiceImpl` 类来实现 `ConsultantService` 接口，并在里面写具体的业务逻辑代码。  
但是在`Langchain4J`中，我们不需要实现类：

*   **`ConsultantService` 接口**：只是一个“契约”或“遥控器”。它定义了你能做什么（比如 `chat`），但没定义怎么做。
*   **`AiServices`**：它是“工厂”。它读取你的接口，利用 Java 的**动态代理**技术，在内存中实时生成了一个实现了该接口的对象。

## 共有内容

当我想实现一个简单的`AIService`，有些内容是通用的，分别是：

1.  添加依赖

1  
2  
3  
4  
5  
6  
7  
8  
9  
10  

<dependency>    
    <groupId>dev.langchain4j</groupId>    
    <artifactId>langchain4j-open-ai-spring-boot4-starter</artifactId>    
    <version>1.14.0\-beta24</version>    
</dependency>    
<dependency>    
    <groupId>dev.langchain4j</groupId>    
    <artifactId>langchain4j-spring-boot4-starter</artifactId>    
    <version>1.14.0\-beta24</version>    
</dependency>  

2.  定义接口

1  
2  
3  
4  
5  
6  
7  

package org.example.cosultant.aiservice;    
    
import dev.langchain4j.service.spring.AiService;    
import dev.langchain4j.service.spring.AiServiceWiringMode;    
public interface ConsultantService {    
    public String chat(String message);    
}  

3.  注入使用

1  
2  
3  
4  
5  
6  
7  
8  
9  
10  
11  
12  
13  
14  
15  
16  
17  
18  

package org.example.cosultant.controller;    
    
import dev.langchain4j.model.openai.OpenAiChatModel;    
import org.example.cosultant.aiservice.ConsultantService;    
import org.springframework.beans.factory.annotation.Autowired;    
import org.springframework.web.bind.annotation.RequestMapping;    
import org.springframework.web.bind.annotation.RestController;    
    
@RestController    
public class ChatController {    
    @Autowired    
    private ConsultantService consultantService;    
    @RequestMapping("/chat")    
    public String chat(String message){    
        String result \= consultantService.chat(message);    
        return result;    
    }    
}  

区别在于编程式的方法，在定义完接口后，需要通过`AIService`在后台创建一个隐形的类，这个类的 `chat` 方法里封装了调用 OpenAI 接口、处理上下文、解析结果的所有复杂逻辑。然后，它把这个**代理对象**注册到了 Spring 容器中。

## 为什么需要这两种集成方式

在Java生态中，LangChain4j作为最成熟的LLM应用框架，与Spring Boot的集成提供了两种主流开发范式：  
●**编程式（Programming Mode）**：通过`AiServices.builder()`手动构建AI服务  
●**声明式（Declarative Mode）**：通过`@AiService`注解自动装配AI服务  
两种方式都能实现AI服务与Spring容器的集成，但设计哲学和实现细节存在本质差异。本文将从**技术原理**、**代码结构**、**适用场景**三个维度进行深度对比。

## 原理剖析

### 编程式

**实现原理**：

1.  开发者显式调用`AiServices.builder()`，通过链式调用指定`chatModel`、`chatMemory`等依赖
2.  在`@Configuration`类中通过`@Bean`将构建好的服务注册到Spring容器
3.  本质是**工厂方法模式**，开发者完全控制对象的创建过程  
    **代码示例**：

1  
2  
3  
4  
5  
6  
7  
8  
9  
10  
11  
12  
13  
14  
15  
16  
17  
18  
19  
20  
21  
22  
23  
24  

package org.example.cosultant.config;    
    
import dev.langchain4j.model.openai.OpenAiChatModel;    
import dev.langchain4j.service.AiServices;    
import org.example.cosultant.aiservice.ConsultantService;    
import org.springframework.beans.factory.annotation.Autowired;    
import org.springframework.context.annotation.Bean;    
import org.springframework.context.annotation.Configuration;    
    
@Configuration    
public class CommonConfig {    
    @Autowired    
    private OpenAiChatModel model;    
   %%1. AiServices 读取 ConsultantService 接口的定义    
     2. 它将接口与你配置好的 OpenAiChatModel (大模型) 绑定    
     3. .build() 方法会在内存中生成一个“代理对象”  %%   
    @Bean    
    public ConsultantService consultantService(){    
        ConsultantService consultantService= AiServices.builder(ConsultantService.class)    
                .chatModel(model)    
                .build();    
        return consultantService;    
    }    
}  

**技术特点**：  
这种技术的缺点很明显，代码冗余，需维护配置类与接口的绑定关系，作为小白的我一下子是看不明白为啥要这么写的，但是优点也很明显，就是灵活性高。  
当我们这么写了以后，Spring 启动，扫描 `CommonConfig` 类，发现`CommonConfig`里有一个字段`model`贴着`@Autowired` 标签。Spring 去自己的容器（BeanFactory）里找类型为 `OpenAiChatModel` 的对象，LangChain4j 自动配置类早就创建了一个 `OpenAiChatModel` 的 Bean 放在容器里了，Spring 通过**反射机制**，强行把找到的那个 Bean 赋值给 `private` 字段 `model`（即使它是私有的，Spring 也能通过 `setAccessible(true)` 修改它），然后我们在`consultantService()`方法就能直接使用`model`了。  
再来说为何`consultantService()`要这么写，我们刚才定义了一个没有实体的接口，如果要使用它，就要把它**变成一个能干活并注册到 Spring 容器里的 Bean**。  
在方法前面写`@Bean`是告诉 Spring 容器，“这个方法返回的对象，请你帮我管理起来”。我们在`controller`方法中用`AutoWired`想要注入`ConsultantService`。如果没有这个`@Bean`，Spring 容器里就没有这个服务的实例，启动时就会报错。然而`ConsultantService`只是一个`interface`,接口是不能直接`new`的，通常我们需要写一个 `ConsultantServiceImpl` 类来实现它。  
但是`Langchain4J`提供了一种代理注入的方法，通过:

1  
2  
3  

ConsultantService consultantService= AiServices.builder(ConsultantService.class)    
                .chatModel(model)    
                .build();   

上述方法它在内存中动态生成了一个“代理对象”（JDK 动态代理）。- 这个代理对象实现了 `ConsultantService` 接口。- 它的内部逻辑被设定为：收到调用 -> 发给 AI -> 返回结果。  
这段代码利用**工厂模式**和**动态代理**，帮我们一行代码就搞定了所有这些复杂的实现逻辑，否则我们需要手动去写一个 `ConsultantServiceImpl` 类，并在里面写几十行代码来处理 HTTP 请求、上下文记忆和 AI 调用。

## 声明式

如果我们考虑使用声明式的方法调用`AIService`，只需要在`ConsultantService`接口声明的上面，添加注解:

1  
2  
3  
4  
5  

    
@AiService(    
        wiringMode = AiServiceWiringMode.EXPLICIT,//手动装配    
        chatModel = "openAiChatModel"//AI模型    
)  

这样框架在启动时自动扫描并生成代理对象，依赖通过注解属性（如`chatModel`）显式指定，或默认自动注入。  
这样就不需要`@Bean`直接调用`ConsultantService`接口实现的方法了。

## 自我启发

由于刚开始学习Java和Langchain4J，很多常见的设计模式我都不懂，通过两种不同的方法，使用`AIService`,我理解到了自身很多的缺点和需要补充的，具体如下：

*   **Spring AOP动态代理学习与理解**
*   **工厂方法模式理解**
*   **Bean工作流程理解**  
    后续会在学习Langchain4J的基础上，有意识的进行补充和学习。
