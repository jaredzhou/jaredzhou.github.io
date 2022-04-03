---
title: "任务流"
date: 2021-09-08T21:33:24+08:00

comment: true
mathjax: false
toc: true
autoCollapseToc: true
draft: false
---

### 需求描述

* 由于前端展示的多样性， 需要一个可以灵活配置的Api Gateway, 这个gateway不能仅仅满足http请求都后端微服务的转发， 还必须可以定义微服务返回结果之间的组合，组合的灵活性当然是不可能达到代码的程度，但是应该可以满足常见的数据拼接，比如结果集的简单Merge，比如数据集详情扩展， 比如B接口请求依赖A接口的返回值等。
* 要能够请求多个接口，并且这些请求之间有一定的依赖关系， DAG是一个自然的选择， 如果把每个请求当成单个节点上的任务, 那么任务之间的依赖关系可以通过节点依赖关系表达出来， 同时没有依赖关系的任务还可以并行执行。
* 一个DAG可以表示一个任务流，一个任务流应该有Request， Response， 一个任务流应该有开始节点与结束节点。一个DAG应该允许节点中存在SubDAG
* 任务之前必须可以传递数据，我们定义A->B为B依赖A， 那么应该可以将A的执行结果传递给B。如果存在依赖关系{A->B, C->B},那么应该可以将A的结果合并C的结果。
* 由于在Gateway中使用， 任务的主要实现应该为 Http请求， RPCX请求， GRPC请求等， 一个问题在于如何将客户端的Http请求转化为后端服务请求，通常的情况是， 一个Http请求的query string或者body加上一些用户信息传递到后端服务，但是由于接口是从已经开发完成的系统中迁移， 所以情况可能会较为复杂。Query String, Params, Header, Body, JWT,这个都可能是参数的来源
* 认证与授权也可以当作节点来使用
* 如果任意一个节点失败， 那么请求应该是失败的， 并且返回错误信息给前端

### 基本设计
* Request 包含http所有可能的信息， 必须包含一个requestID，假如没有就生成一个
* Response 主要包含body与header， 两个部分， header中必须包含requestID
* 每一个接口(POST /api/username)对应一个DAG， 这个DAG可以使用JSON/YAML配置
* DAG的执行应该是异步的， 每个Node可能在不同的协程中执行，DAG应该有一个Executor， 这个Executor应该负责Node的开始执行， 捕获Node执行完成信号， 并且通知所有满足依赖关系的Node继续执行，直到结束节点执行完毕。 这个Executor应该包含一个Context， 并且在所有Node中共享这个Context
* Node应该在一个协程池当中执行， 一个Node完成后在Context中设置完成结果，并且通过channel通知Executor