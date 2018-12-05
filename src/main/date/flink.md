
## Use Cases
### Event-driven Application
   *  事件驱动的优点
      * 通过访问本地数据以获得高性能
      * 定期checkpoint到远程数据中心
   * Flink如何支持事件驱动
      * 事件驱动程序的限制是 如何处理好时间和状态
      * flink杰出的特征是 savepoints

### Data Analytics Application
   * 分析程序 
      * 有限的数据集，重新运行
      * link支持实时的
   * flink的优点
      * 低延迟
      * 架构优越，容错
   * flink
      * 高层抽象接口处理 batch 和 streaming

### Data Pipeline Applications
   * Period ETL
   

## Flink DataStream API Programming Guide
### Data Sources
   * File-based
   * Socket-based
   * Collection-based
   * Custom
   
### DataStream Transfromations 

### Data Sinks
   * writeAsText
   * writeAsCsv
   * print/printToErr
   * writeUsingOutputFormat
   * writeToSocket
   * addSink
   
### Iterations

### Execution Paramters
   * setAutoWatermarkInterval
   * FaultTolerance
   * Controlling Latency
   
### Debuging
   * Local Execution Environment
   * Collection Data Sources
   * Iterator Data Sink

   
## Envent Time
### Overview
   * defination 
	   * Processing Time: When the data is processed by the operation
	   * Event Time: The time when the event is created in the producer
	   * Ingestion Time:   The time when the event enters the flink
   
   * setting a Time Charcteristic
      * evn.setStreamTimeCharacteristic

### Event Time and Watermarks
   * . A Watermark(t) declares that event time has reached time t in that stream, meaning that there should be no more elements from the stream with a timestamp t’ <= t (i.e. events with timestamps older or equal to the watermark).

### Watermarks in Parallel Streams
   * 乱序问题

### Late Elements
   * allowed lateness
   
### Idling sources
   * no output due to no event
   * use periodic watermark assigners that don't only assign based on element timestamp.

### Debugging Watermarks


### How operators are processing watermarks
   * 通用规则：operators在将时间传递给下游之前，需要处理完给定watermark之前的事件

### Generating Timestamps / Watermarks
* Assigning Timestamps
   * Source Functions with Timestamps and Watermarks
      * sourceContext.emitWatermark(new Watermark())
   * TimestampAssigners/Watermark Generators
      * with Periodic Watermarks: AssignerWithPeridoicWatermarks
      * With Punctuated Watermarks: AssignerWithPunctuatedWatermarks

* Timestamps per Kafka Partition
   * AscendingTimestampExtractor

### Pre-defined Timestamp Extractors/Watermark Emitters
* Assigners with ascending timestamps
* Assigners allowing a fixed amount of lateness


## State & Fault Tolerance
### Overview

### Working with State

### The Broadcast State Pattern

### Checkpointing

### Queryable State

### State Backends

### State Schema Evolution

### Custom State Serialization


   





