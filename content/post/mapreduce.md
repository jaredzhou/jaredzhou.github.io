---
title: "MapReduce"
date: 2021-09-25
comment: true
mathjax: false
toc: true
autoCollapseToc: true
draft: false
---
## 系统描述
根据Google论文mapreduce开发一个简易的mapreduce系统，可以处理多个数据文件，最后生成一个结果，课程案例主要是对文本进行WordCount，首先通过map统计每个word出现的次数，然后通过reduce计算count
### 设计
* Coordinator 协调者负责分配任务，包括map任务，reduce任务。
* Worker 执行map， reduce任务， 任务通过rpc请求Cooridnator获取 
* Coordinator要包含错误处理，错误包括任务失败与任务缓慢，处理措施都是重试

### 实现
* Coordinator包含两个任务队列,一个mapTasks，由输入文件数决定； 一个reduceTasks,由reduce数量决定
* 当worker请求任务时， Coordinator会分配一个Ready状态的任务，任务一旦分配就设置成Pending状态。
* Worker完成任务后，通知Coorinator，此Task已完成, 等待1秒后继续请求下一个任务
* 当所有reduceTask都完成后， 整个任务完成

### 错误处理
* 当一个任务pending超过10秒后，如果状态不为Finish， 则重新设置成Ready
* Worker执行任务失败后，直接结束当前循环，等待下一个任务请求
* 任务执行成功后， Coordinator会对请求任务的worker返回Done标记， worker接收标记后退出；如果Coordinator已经退出， 则网络连接失败， Worker退出；其他情况Worker会一直循环请求任务
* Map Worker将TempFile作为中间数据， 并将这些文件传输给Master，以供Reduce Worker使用。一个已经成功返回的MapTask将忽略其他的返回
* Reduce Worker会使用TempFile作为中间输出，当前任务完成后，Rename这些文件到正确的文件目录，所以当Worker执行失败时这些文件并不可见


### 流程图
![](arch.png)