# kube-dev-book
《kubernetes开发指南》

# 作者简介
方海涛 sealos作者，五年以上容器平台与系统研发经验, 自研kubernetes CNI 对接ovn网络，动态本地存储CSI等。丰富的kubernetes源码阅读与定制扩展经验，融合默认调度器与深度学习批任务调度器。开源golang websocket框架lhttp, kubernetes管理平台fist等。

# 图书信息
## 选题背景,市场需求,产品定位
   云原生的迅速崛起让越来越多开发者进入容器领域，作为云原生的核心项目kubernetes更是从业者最需要掌握的一门技术，市场上关于使用和入门类书籍非常之多，但是开发类书籍基本还是一个空缺，同样社区的官方文档也少之又少。本书希望为开发提供一定的指导作用，在开发中少走弯路，适合已经具备一定容器技术基础和编程能力基础的读者阅读，属于kubernetes进阶类书籍。

## 内容简介
   kubernetes具备极强的扩展性，本书通过深入浅出的方式介绍如何开发kubernetes, 包含上层的client-go使用，CRD开发，adminssion webhook, 和底层三大接口实现开发（CNI CSI CRI），以及核心组件的定制，如定制调度器，apiserver,kubelet等。通过学习这些开发技巧能更深入的理解技术原理以及在系统之上实现我们自己想要的功能。

## 大纲目录
1. 开发环境与编译测试
    1.1. 编译kubernetes源码
    1.2. CI/CD自动化编译源码
    1.3. Makefile与编译脚本分析
2. client-go开发指南
    2.1. 环境构建
    2.2. 一个简单的示例
    2.3. client-go架构分析
    2.4. dynamic client
    2.5. rest client
    2.6. cache client
    2.7. informal
3. CRD与adminssion webhook开发
    3.1. 一个Cronjob示例
    3.2. API版本
    3.3. CRD编码
        3.3.1. 生成CRD
        3.3.2. 使用Finalizers
        3.3.3. adminssion webhook
        3.3.4. 代码生成标记
        3.3.5. contoller生成命令行工具
        3.3.6. 编写测试用例
        3.3.7. metrics
4. CRI实现
5. CNI实现
6. CSI实现
7. 扩展apiserver
8. 定制kube-sheduler
9. 定制kubelet

## 本书特色
> 全面

  本书从上层开发到底层云内核（指kubernetes）云驱动（指CNI CSI CRI等）开发都会涉及。

> 实用
 
   书中内容是kubernetes开发必然会涉及到的一些主流方法，能够帮助我们快速实现一些想要的功能，以及在原生特性不满足时如何进行改造扩展。

> 注重原理

   很多一些细节知识，如某个接口某个函数是会经常发生变化，所以本书更侧重透过代码看本质，看原理理念与标准等，细节东西读者可能会忘记，但是这些理念性的东西会永远沉淀下来成为读者的宝贵财富。

> 不乏实践

   书中开发例子会针对一些特定场景来设计一些demo，我们鼓励读者多实践从而加深对原理的理解，案例代码我们也会放到github上进行长期更新，书中更多设计一些重要逻辑。

## 重要信息

读完本书读者应当掌握以下内容：

1. 熟练使用client-go进行上层开发，理解client中的cache informal等原理
2. 熟悉CRD adminsion webhook开发，并深入掌握一个CRD框架的使用与原理
3. 熟悉各核心组建的原理并理解整体框架，具备扩展任何一个核心组件的能力，理解核心组件中的“套路”，以不变应万变
4. 掌握自研CNI CSI CRI的能力
