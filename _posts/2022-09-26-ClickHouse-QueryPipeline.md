---
layout: post
title:  "ClickHouse QueryPipeline 相关代码分析"
date:   2022-09-26 00:08 +0800
categories: ClickHouse PipelineExecution
---

## Overview

QueryPipeline 是 CK 中一条 select query 最终的物理执行计划。本文尝试分析 QueryPipeline 的创建过程，一些相关结构之间的关联，以及各个结构所起到的作用。

本文基于 22.3.7.28-lts 版本分析。
## QueryPipeline 创建

首先在 InterpreterSelectQuery 的 execute() 中，调用 query_plan.buildQueryPipeline 来生成 QueryPipelineBuilder。再通过 QueryPipelineBuilder::getPipeline(QueryPipelineBuilder) 来创建真正的 pipeline。

query_plan.buildQueryPipeline 首先会执行 query plan 的 optimize，之后从 source step 开始，调用每个 QueryPlanStep 的 updatePipeline，以上一个 step 生成 pipelines 作为参数，生成当前 step 的 QueryPipelineBuilder，并将其设置为 last_pipeline。等所有的 step 执行完 updatePipeline 之后，返回最后的 last_pipeline，也就是最终的 QueryPipelineBuilder。

QueryPipelineBuilder::getPipeline 中，会基于参数得 QueryPipelineBuilder 的 pipe 创建来QueryPipeline。QueryPipeline 构造时，如果发现传入得 pipe 的 numOutputPorts 大于 0， 则添加一个 ResizeProcessor，将所有的 outputs 都接到这个 ResizeProcessor 上，确保到这里多个线程的执行结果都汇总到一起。

需要注意的是，interpreter 返回结果中的 pipeline，不是最终的 pipeline，还需要以下的两步：

- 在 executeQuery.cpp 的  executeQueryImpl 中，对于 pulling 的 pipeline，会在最后添加一个 LimitsCheckingTransform processor，用于进行 time limit 和 size limit 的判断，以及执行信息的统计（result_rows, result_bytes, execution_time）
- 在 PullingAsyncPipelineExecutor 的构建时，会加上  LazyOutputFormat processor

至此 pipeline 才算构建完成。

## Pipe 创建

QueryPipelineBuilder 中会包含一个 pipe 结构，这个结构主要用于进行 processors 之间的连接，是 plan 到 pipeline 转换过程中的关键结构。这里分析 pipe 的创建以及关键的连接操作。

创建 pipe 时，首先通过所有的 main Sources processors 创建。

pipe 在添加 sources 时，是每个 source 创建一个 Pipe，然后再调用 unitePipes 将所有的 Pipes 连接成一个。

### Pipe construction

1 将 source 的 outputs 的 front 添加到 output_ports 中

2 将 header 设置为 output_ports.front() 的 header

3 将 source processor 添加到 processors 中

4 设置 max_parallel_streams 为 1

### unitePipes

1 创建一个新的 Pipe res

2 res 的 holder 设置为最后一个 pipe 的 holder

3 res 的 header 设置为 pipes 的 common header

4 将所有 pipes 的 processors 添加到 res 的 processors 中

5 将所有 pipes 的 output_ports 添加到 res 的 output_ports 中

6 累加所有 pipes 的 max_parallel_streams，赋值给 res 的 max_parallel_streams

对于 transform processor 的添加，pipe 提供两种接口，其中 addSimpleTransform 接收一个 getter 的 transform processor 创建方法，为每个 output 创建一个对应的 transform processor，并进行连接；addTransform 直接接受一个 transform processor，将所有的 outputs 都和这一个 transform processor 进行连接。

### addSimpleTransform(const ProcessorGetter & getter)

将 transform processor 添加给所有的 output_ports。

具体来说，对每个 output_port：

1 基于传入的 getter 函数，创建 transform processor

- 基于传入的 input header 和 output header 创建 input port 和 output port

2 将当前 output port 和 transform 的 input port 做 connect

3 将这个 output port 设置为 transform 的 output port

4 processors 中添加 transform

### **addTransform(ProcessorPtr transform, OutputPort * totals, OutputPort * extremes)**

将传进来的 transform processor inputs 和所有的 output ports 连接

1 获取 transform 所有的 input ports

2 将每个 input port 和 output ports 顺序连接

3 将 outputs 设置为 transform 的 outputs

4 将 outputs 中的每个 元素，都添加到 pipe 的 output_ports 中

5 将 header 设置为 output_ports 第一个的 header

6 将 transform processor 添加到 processors 中

7 设置 max_parallel_streams 为当前 max_parallel_streams 与 output_parts.size() 的最大值

## 其他几个问题

### processors 如何进行连接

- 每个 processor 在创建时，会根据 input header 和 output header 来创建 input 以及 output ports。每次 pipe 添加 processor 时，会将当前的 output ports 和 processor 的 input port 连接，具体操作为：
    - 分别设置为对方的 output_port 以及 input_port
    - 分别设置 out_name 与 in_name 为 output 和 input processor 的 name
    - 初始化 input.state, output.state 设置为 input.state

### 如果多个 processors 的 output 是同一个 processor， ExecutingGraph 如何调度

- 对于 inputs 来说，每个 processor 都会将 output processor 添加到 edges 中，然后调用一次 output processor 的 prepare 方法。这种汇聚 inputs 的 processor 一般是一个 ResizeProcessor，在 prepare 的时候，只负责将 input 的数据 push 到 output processor 去，然后继续返回 needData 的状态。

### 多个 sources 是在什么时候合并的？是否有固定的合并操作？

- QueryPipeline 构造时，引入 ResizeProcessor，进行多线程 inputs 的合并，之后就都是单线程操作

### ISource 的 outputs 如何初始化？

ISource 的 construction：

```cpp
ISource::ISource(Block header)
    : IProcessor({}, {std::move(header)}), output(outputs.front())
{
}
```

用的是 IProcessor 创建时候的 outputs 的第一个，而 IProcessor 的 outputs 是基于 header 创建的， 这里主要是创建了一个带 header 的 OutputPort。header 来自于

MergeTreeBaseSelectProcessor::transformHeader 操作，该操作会基于 required columns 等信息算出来的需要实际读取的 columns，也就是 OutputPort 的 header 记录的是 ISource 要读取的列名称。