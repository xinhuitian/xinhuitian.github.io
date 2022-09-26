---
layout: post
title:  "ClickHouse ExecutingGraph 相关代码分析"
date:   2022-09-26 00:08 +0800
categories: ClickHouse PipelineExecution
---

## Overview

每条 select query 在生成完 Processors 之后，准备执行时，都会创建一个 PullingAsyncPipelineExecutor 结构来负责整体 pipeline 的 pull 式执行，其中 pipeline 的执行是被封装在一个 PipelineExecutor 结构中管理，而 PipelineExecutor 为每个 Pipeline 的 Processors 维护了两个主要的数据结构来管理整体的执行，一个是负责管理各个 Processors 之间的调用关系和状态的 ExecutingGraph，另一个是负责具体执行的 ExecutorTasks。本文主要介绍 ExecutingGraph 的基本结构与主要方法设计。

## 基本结构

Node：每个 processor 建立一个，负责维护 Processor 的执行状态

- updated_output_ports
- updated_input_ports

Edge：

- to：连接 node 的 id
- input_port_number：这个 output port 对应的 input port 在 to Processor 的 inputs 中的 port number
- output_port_number：output port 在 from Processor 的 outputs 中的 port number
- update_info: Port 会操作的信息，edge 会将 update_info 的 id 设置为自己，会将 update_info 的 update_list 设置为 node 的 post_updated_input_ports (back edges) 或者是 post_updated_output_ports (direct edges)

processors_map：IProcessor 到 node 的 id 的 map

## Construction

基于给定的 processors，包含创建 nodes 和创建 edges 两个步骤。

### 创建 nodes

针对给定的每个 processor，都创建一个 node，将 processor 和 node id 的对应关系保存到 processors_map 中。

### 创建 edges

对每个 node，调用 addEdges 方法，为每个 node 创建 edges

## addEdges

对一个 node 来说，首先添加其对应 processor 的 inputs，再添加其 outputs

添加 inputs / outputs

- 对该 node 对应的每个 input port / output port，获取其对应的 output port / input port的 processor 作为 to processor
- 得到 output port / input port 在 processor 中的 output port number / input port number
- 创建 edge: 
  ```Edge edge(0, true, from_input, output_port_number, &nodes[node]->post_updated_input_ports); / Edge edge(0, false, input_port_number, from_output, &nodes[node]->post_updated_output_ports);`
- 调用 addEdge：
    - 首先查找 to processor 是否已经在 processors_map 中了，如果不在，就会 throw exception
    - 将 edge→to 设置为 to processor 对应的 node
    - 将 edge 添加到 nodes[node]→back_edges / nodes[node]→direct_edges 结构中
    - 将 edge 的 update_info.id 设置为 edge 自己
- from_input 对应的 input port 设置 update info 为 edge 的 update_info

## initializeExecution

选出所有的没有 outputs 的 processor，也就是所有的 IOutputFormat processors 添加到 stack 中，将 node 的 status 设置为

```
ExecutingGraph::ExecStatus::Preparing
```

对于每个 stack 中的 processor，调用 updateNode

## updateNode

参数为 processor_id, queue, 以及 async_queue。processor_id 表示要更新哪个 node；queue 是一个同步执行的 task queue，updateNode 方法会将 ready 的 node 添加到这个 queue 中，之后由 ExecutorTasks 进行处理；async_queue 和 queue 的使用场景类似，只是只会处理 status 为 async 的 processors。

updateNode 方法在 initializeExecution 以及每个 processor 的 work 执行后会被调用，主要负责 processors 的调用前准备工作，以及 processor node 执行的调度。

对于 IOutputFormat Processor，第一次调用 prepare，会进行以下操作

```cpp
/// 会将其 inputs 添加到 post_updated_input_ports 中
input.setNeeded();
if (!input.hasData())
    return Status::NeedData;
```

对 node post_updated_input_ports 中的每个 edge，添加到 updated_edges 中，之后调用 edge→update_info.trigger() 来将 update_info 的 version+1

继续进入下一层循环，这时 updated_processors 为 empty

- 获取 updated_edges 最上面的 edge
- 得到 edge→to 指向的 node，这里应该是一条 back_edge, 所以指向的是之前 node 的一个 input
- 将 edge 的 output_port_number 添加到 node.updated_output_ports 中
- 如果 node 的 status 为 idle：
    - 将 status 设置为 ExecutingGraph::ExecStatus::Preparing
    - 将 edge→to 添加到 updated_processors 中

继续进行时，又进入到 update_processors 不为空的逻辑中。

如此循环，直到 updated_edges 和 updated_processors 都为空位置。

由此可见，第一次调用 processors 的 prepare 是从后往前调用的。

全部执行完之后，queue 中应该只包含 source processors, 并且这些 node 的 status 都会被设置为

ExecutingGraph::ExecStatus::Executing，其他 processors 会被设置为 ExecutingGraph::ExecStatus::Idle

updateNode 在每个 processor 执行完 work 以后，还会被调用一次，只不过这次应该是从前往后调用。