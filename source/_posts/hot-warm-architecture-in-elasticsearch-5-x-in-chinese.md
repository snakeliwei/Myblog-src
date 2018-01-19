title: Hot warm architecture in elasticsearch
tags:
  - elasticsearch
  - linux
categories:
  - Study
cc: true
hljs: true
comments: true
date: 2018-01-19 16:13:51
---

# [原文链接](https://www.elastic.co/blog/hot-warm-architecture-in-elasticsearch-5-x)

# Elasticsearch 5.x 版本中的冷热节点架构
当Elasticsearch用于大量实时数据分析的场景时，我们推荐使用基于时间的索引然后使用三种不同类型的节点（Master, Hot-Node 和 Warm-Node）进行结构分层，这就是所谓的"Hot-Warm"架构。每种节点有自己的任务，下面会进行介绍。

### Master 节点
我们推荐每个集群运行三个专用的Master节点来提供最好的弹性。使用时，你还需要将 discovery.zen.minimum_master_nodes setting 参数设置为2，以免出现脑裂的情况。用三个专用的Master节点，专门负责处理集群的管理以及加强状态的整体稳定性。因为这三个Master节点不包含数据也不会实际参与搜索以及索引操作，在JVM上它们不用做相同的事，例如繁重的索引或者耗时，资源耗费很大的搜索。因此不太可能会因为垃圾回收而导致停顿。因此，Master节点的CPU，内存以及磁盘配置可以比Data节点少很多的。

### Hot 节点
指定的Data节点会完成集群内所有的索引工作。这些节点同时还会保存近期的一些频繁被查询的索引。由于进行索引非常耗费CPU和IO，因此这些节点的服务器需要强大的SSD存储来支撑。我们推荐部署最小化的三个Hot节点来保证高可用性。根据近期需要收集以及查询的数据量，可以增加服务器数量来获得想要的性能。

### Warm 节点
这种类型的节点是为了处理大量的而且不经常访问的只读索引而设计的。由于这些索引是只读的，Warm节点倾向于挂载大量磁盘（普通磁盘）来替代SSD。跟Hot节点一样，我们建议部署最小化的三个Warn节点来保证高可用性。然后跟之前一样地，数据量大的话还是需要额外的节点来达到性能要求。而且还需注意的是CPU和内存配置跟Hot节点保持一致。通过测试一些类似生产环境中耗费比较大的查询可以确认这些东西。

Elasticsearch集群需要知道哪些服务器有Hot节点以及哪些服务器有Warm节点。这个可以通过分配所需的[属性](https://www.elastic.co/guide/en/elasticsearch/reference/5.1/allocation-awareness.html#forced-awareness)给服务器来实现。

例如，你可以在 elasticsearch.yml 这个配置文件中通过 node.attr.box_type: hot 把节点设置为hot，或者你也可以在启动节点时使用 ./bin/elasticsearch -Enode.attr.box_type=hot 参数指定。

box_type 这个属性字段你完全可以自定义成你要的。这些自定义的值用于告知 Elasticsearch 从哪里分配索引。

通过以下配置创建索引，我们可以确保今天的索引落在使用SSD的Hot节点上：
```
PUT /logs_2016-12-26
{
  "settings": {
    "index.routing.allocation.require.box_type": "hot"
  }
}
```

过几天之后如果索引不再需要性能好的硬件时，我们可以将这些节点标记成Warm属性，更新索引配置如下：
```
PUT /logs_2016-12-26/_settings 
{ 
  "settings": { 
    "index.routing.allocation.require.box_type": "warm"
  } 
}

```

那么现在我们可以使用logstash或者beats来实现：
如果索引模板在logstash或者beats中管理，那么索引模板需要做一些更新，包括分配过滤器。"index.routing.allocation.require.box_type" : "hot" 这个配置会使新的索引创建在Hot节点上。
例如：
```
{
  "template" : "indexname-*",
  "version" : 50001,
  "settings" : {
             "index.routing.allocation.require.box_type": "hot"
 ...
```
另外一个策略是给集群中的所有索引添加一个普通模板，在Hot节点上 "template": "*" 模板可以生成新的索引。
例如：
```
{
  "template" : "*",
  "version" : 50001,
  "settings" : {
           "index.routing.allocation.require.box_type": "hot"
 ...
```

当你确认一个所以不再承担写入以及不需要频繁搜索时，它可以从Hot节点中合并到Warm节点。这个可以通过更新它的索引配置："index.routing.allocation.require.box_type" : "warm" 轻而易举地完成这个操作。
Elasticsearch会自动合并索引到Warm节点。

最后，我们还可以在所有Warm数据节点上开启更好的压缩配置，在elasticsearch.yml配置文件中的 index.codec: best_compression 的这个配置项可以配置。
当数据移动到Warm节点后，我们可以调用 _forcemerge API 来合并分段: 虽然可以节约内存, 磁盘空间以及更少的文件句柄, 也有使用新的 best_compression 编码进行索引重写所带来的副作用. 

当还需要分配到 strong boxes 时强制合并索引不是什么好办法，这些节点上的进程会优先进行I/O操作然后影响到正在进行索引的当天日志。但是 medium boxes 则不会有太多操作，所以这是安全的。
现在我们已经看到如何手动修改索引的分片分配，接下来让我们来看下如何使用[Curator](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/installation.html)这个工具来自动处理这些事情。
下面的例子中我们使用curator 4.2从Hot节点移动三天前的索引到Warm节点：
```
actions:
  1:
    action: allocation
    description: "Apply shard allocation filtering rules to the specified indices"
    options:
      key: box_type
      value: warm
      allocation_type: require
      wait_for_completion: true
      timeout_override:
      continue_if_exception: false
      disable_action: false
    filters:
    - filtertype: pattern
      kind: prefix
      value: logstash-
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y.%m.%d'
      unit: days
      unit_count: 3
```
最后我们可以使用curator来强制合并索引。执行优化之前要确保等待足够长的时间进行索引重新分配。你可以设置操作1中 wait_for_completion，或者修改操作2中的 unit_count 来选择4天前的索引.这样就有机会在强制合并之前完全合并。
```
2:
    action: forcemerge
    description: "Perform a forceMerge on selected indices to 'max_num_segments' per shard"
    options:
      max_num_segments: 1
      delay:
      timeout_override: 21600 
      continue_if_exception: false
      disable_action: false
    filters:
    - filtertype: pattern
      kind: prefix
      value: logstash-
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y.%m.%d'
      unit: days
      unit_count: 3
```
> 注意 timeout_override 要比默认值 21600 秒大，不过它可能会更快或者慢一点，这取决于你的配置。
从Elasticsearch 5.0开始我们还可以使用 Rollover 和 shrink api 来减少分片数量，可以以更简单高效的方式来管理基于时间的索引。你可以在这个[博客](https://www.elastic.co/blog/managing-time-based-indices-efficiently)中找到更多细节。
