---
title: 如何零基础快速拥有一个自己的Springboot项目
date: 2026-05-07
tags:
  - Springboot
---

## 前言

最近入职后准备使用`Langchain4J`对`Java`平台开发的项目进行`Agent`的融合，支持一些自动化`Agent`运营的功能。由于之前完全没有使用过`Java`，所以用了两天时间熟悉了下什么是`Springboot`,但是依旧很模糊，只做到了启动项目的过程。  
在这几天的熟悉`Langchain4J`的过程中,通过使用`Springboot`我对它有了一些基础的理解，在此作为记录。

## 一些通识的东西

关于`Springboot`,我们可以用大模型等等了解它相关的很多东西，这些并不是我们关注的重点，但是如果想快速拥有一个属于自己的`Springboot`项目，一些常识是需要知道的。

### 如何管理相关依赖

Java项目的依赖配置文件统一是在`pom.xml`文件中做管理，如果我们在路径中引入了`spring-boot-starter-web`）和现有的配置，自动为 Spring 应用程序配置所需的 Bean 和组件。例如，当我们引入了 Web 相关的依赖，Spring Boot 就会自动配置好 Spring MVC、内嵌的 Tomcat 服务器等，这样无需手动编写任何 XML 或 Java 配置。这样可以做到快速的启动一个Java项目（使用IDE的`maven`刷新下即可）。

### 如何管理相关配置

Java项目的配置文件，一般是在`application.yml`中进行设置的,默认用IDE启动的`springboot`项目会有一个文件是`application.properties`，它的特点是传统，键值对的形式，但是没有用`application.yml`来的更加的结构清晰。  
如果我们想要使用它，可以通过`application.yml`去做配置管理，比如端口、密码、密钥，以及做生产或者开发环境的配置切换。

### 如何管理相关前端代码

前端的代码我们可以在`resource/static`这个目录下写一些基础的`index.html`,更复杂的React目前我并不会，但是也能满足基础的需求（后续可以研究？）。

## 什么是Entity

所谓的实体层，也就是`model`层，它是数据的载体，对应的就是数据库中的表。一个`entity`类通常对应数据库的一张表，类中的方法对应表的字段。纯数据的对象一般只包含属性字段的`getter`和`setter`方法，不包含业务逻辑。常用的注解有`@Entity`,`@Table`,`@Id`,`@Column`等

## 什么是DAO

当我们定义了数据的实体后，肯定需要通过一些增删改查对数据库进行操作，这个时候就需要`DAO/Mapper`层进行数据的一些操作，常用的注解有`Repository`或者`Mapper(MyBatis)`。这些基础的`CRUD`操作也可以用大模型去生成写作，不过需要核对的是存取数据的一些口径（时间、周期）等，在AI刚兴起的时候我做产品吃过这个亏，程序员写的代码取数逻辑有问题，折磨了我很久很久。

## 什么是Sevicce

接下来就是最关键的`Service`层，通常采用接口+实现类的模式。

*   接口：位于`..service`包下，充当说明书，只定义业务方法的名字，但不写具体的代码。
*   实现类：位于`..service.impl`包下，是真正干活的地方，它会`implements xxService`然后写具体的业务。  
    为什么我们要在`Service`层写接口，是为了解耦，这样`Controller`层在调用`Service`层的时候，是面向接口编程，如果要修改业务逻辑，只需要新建一个实现类，而不需要修改`Controller`的代码。  
    在标准的 Spring 开发中，如果你写了一个接口 `ConsultantService`，但没有提供它的实现类（比如 `ConsultantServiceImpl`），Spring 容器在启动时会报错。因为 Spring 不知道应该为这个接口创建什么样的 Bean（对象）来注入。  
    以下代码是没有写`impl`的接口，而是通过注解来实现动态代理为我们生成实现类。

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

package org.example.cosultant.aiservice;    
    
import dev.langchain4j.service.spring.AiService;    
import dev.langchain4j.service.spring.AiServiceWiringMode;    
    
@AiService(    
        wiringMode = AiServiceWiringMode.EXPLICIT,//手动装配    
        chatModel = "openAiChatModel"//AI模型    
)    
public interface ConsultantService {    
    public String chat(String message);    
}  

当我们使用`@AiService`的时候，它会告诉`Langchain4J`框架，为我们的接口自动生成一个实现类，并把它注册到`Spring`容器中。这个过程具体如下：

*   启动扫描：当`Spring Boot` 应用启动时，`LangChain4j` 的自动配置模块会扫描所有带有 `@AiService` 注解的接口。
*   创建代理：框架会利用 Java 的反射和动态代理技术，在内存中动态地创建一个实现了 `ConsultantService` 接口的代理对象（Proxy Object）。
*   注册Bean：这个代理对象会被当做一个普通的 Bean，自动注册到 Spring 容器中。
*   成功注入：当你在 Controller 或其他地方使用 `@Autowired` 时，Spring 会从容器中找到这个由 LangChain4j 创建的代理 Bean，并成功注入。

## 什么是Controller

在我们的`Controller`层，会引入`Service`层定义的接口，并写系统与外界交互的代码，比如接受请求、参数校验、调用业务和返回结果。在这个过程中，有一些通用的操作，比如使用典型的注解：`@RestController`, `@RequestMapping`, `@GetMapping`, `@PostMapping`。  
`@RestController`的作用就是告诉`Spring`这个类是一个控制器，它里面的所有方法都直接返回数据，而不需要跳转页面。  
以下是一个基础的`Controller`层代码，它有一个区别就是有个`Autowired`这里面涉及了动态代理的技术。Spring 只能注入它管理的 Bean，而 Bean 通常是一个具体的类（Class）的实例，接口（Interface）本身是无法被实例化的。

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

这个`@Autowired`并不是接口本身，而是动态代理的实现。当我们使用如下代码的时候,实际上是把容器中已经存在的代理对象拿出来赋值给`consultantService`这个变量（因为启动的时候已经把代理对象放在容器里了）

1  
2  

@Autowired    
    private ConsultantService consultantService;   

当我们调用传入参数后，就会调用代理对象的方法，来完成一系列的包装、发送、解包、返回操作。

## 总结

目前我对`Springboot`有了一个初步的了解，但是实际的工程代码量是比较少的，因此我需要更多的干中学，勤思考多总结，AI时代来势汹汹，只有让自己成为一个全方面及格的多面体，才能去胜任企业的工作，加油！
