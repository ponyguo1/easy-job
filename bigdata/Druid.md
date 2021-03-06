## Druid

Druid是一个为大型(设计为PB级别)**冷数据集**上实时探索查询而设计的开源数据分析和存储系统,提供极具成本效益并且永远在线的实时数据摄入和任意数据处理,并且在面对代码部署,机器故障以及其他意外情况时能保证系统集群正常运行.

- **列式存储**
- **倒排索引(基于Bitmap实现)**
- **分布式,无共享体系架构**
- **亚秒延迟随机探查十亿行表的索引结构**
- **支持快速聚合,灵活过滤,低延迟**
- **高可用**
- **多租户**
- **实时/批量摄入**
- **基于HyperLogLog近似计算**

### 问题定义

设计目的:用于解决数据摄入和探查大批量事务性事件(log data),这种形式的时间序列数据通常存在与联机分析处理(OLAP)工作流中.

### 适用场景

- 清洗好的记录实时录入,不支持更新操作
- 支持宽表,不用Join方式
- 时区和时间维度要求高的场景特别适合

### 架构

Druid集群由不同类型的节点组成,每个类型的节点被设计用来执行一组特定的任务,简化整个系统的复杂性,不同节点类型操作相对独立,并且不同节点类型之间交互极少.因此集群内通讯故障对数据可用性影响极小.

![Druid 集群和集群中的数据流概览](../images/Druid 集群和集群中的数据流概览.jpeg)

1. **Real-Time**节点

   1. **负责摄入和查询事件流**:经过该节点的事件被实时索引,且对查询即时可用.节点只关注一段较短时间范围内的有效事件,并且定期将其在这段较短时间范围内收集到的一批批不可变事件,传送给集群中专门处理批量不可变事件的其他节点.Real-Time节点利用Zookeeper来与集群中剩余的其他节点协调配合.所有节点在Zookeeper中通知它们的在线状态和它们所提供的数据.

   2. 所有流入事件维护一个**内存索引缓冲器(index buffer)**,当事件摄入,且索引可以被直接查询时,存在于缓冲器中的索引将会增量更新.该节点会周期性地或者达到最大行数限制后,将其持久化到硬盘,为了使持久化后数据仍然能够被查询,会将其装载到堆外内存,在传递出去之前会被定期合并,查询将会同时命中内存中或者已持久化的索引.

      ![维护内存索引缓冲器](../images/维护内存索引缓冲器.png)

   3. Real-time节点周期性的执行一个用以搜索本地已持久化的索引的后台任务.该任务合并上述索引,然后创建一个不可变数据块,该数据块包含着某个时间跨度范围内的所有已被Real-time节点摄入的事件,这个数据块为一个"**Segment**",数据传输阶段会将这个Segment上传到永久备份存储中,为一个分布式文件系统(HDFS,S3)中,又被称为"**Deep Storage**".

   4. 节点启动,摄入数据,节点通告其正在提供一个时间区间的数据块,然后每一个持久化周期,节点会将内存中缓冲的数据持久化到硬盘,并刷新内存缓冲区,等待一个可配置的窗口期后,待时间区间里延后的事件大部分都到达,节点合并所有已持久化的索引,然后变成一个不可变的数据块（Segement）,将其分发出去,一旦这个Segment被成功加载,Druid集群其他地方可查,那么Real-time节点将刷新数据,同时通告它正在提供这些数据.

      ![Druid节点启动,摄入数据,持久化,定期传递数据](../images/Druid节点启动,摄入数据,持久化,定期传递数据.png)

   5. **可用性与可扩展性**

      Real-time节点作为数据消费者,需要有响应的生产者提供数据流,因此在生产者和Real-time节点之间部署类似Kafka的消息总线,节点从消息总线中读取事件来摄入数据.

      目的:

      1. 消息总线作为流入事件的缓冲器,Real-time节点在每次持久化它们内存缓冲器中数据到硬盘后更新偏移量,如果节点失败,重启后可以从最近提交的偏移量开始摄入事件极大的缩短了节点恢复的时间.
      2. 多个Real-time节点可以从同一个端点读取事件,从而创建事件副本.一个单一的数据摄入端点也同样允许数据流分片,从而使多个Real-time节点每次摄入一个数据分片,增大吞吐量.

      ![](../images/Real-Time 节点从消息总线中读取事件摄入数据.png)

2. **Historical**节点

   - 加载,提供,删除Real-time节点创建的不可变数据块Segment
   - 无共享架构,无公共资源竞争,节点间互不知晓
   - 支持一致性读：处理不可变数据
   - 并行扫描和聚合不可变数据块,非阻塞

   1. Historical节点在Zookeeper中通告他们的在线状态以及提供的数据,加载和删除Segment指令通过Zookeeper发送,指令包含了Segment在Deep Storage中的位置以及如何解压和处理Segment.Historical节点从Deep Storage中下载某个指定的Segment之前,先检查本地缓存,缓存中维护了Segment在当前节点的信息,如果Segment不存在本地,Historical节点将会从Deep Storage中下载上述指定的Segment,一旦下载执行完成后,Segment相关信息就会在Zookeeper中通告,Segment即可被查询,本地缓存也允许Historical节点快速更新和重启,节点初始化时,节点会检索其资深缓存的所有数据并立即提供数据服务.

      ![](../images/Historical节点下载Segment数据块并载入内存.png)

   2. **层**

      分层作用:根据Segment重要性,将数据设定为更高或者更低的优先级分发.

      1. CPU核更多,内存更大的Historical节点提升到**hot tier**,用以下载更经常被访问的数据,更少的强大硬件来创建冷数据集群,用以存放不经常被访问的Segment.

   3. **可用性**

      Historical节点的Segment加载和卸载依赖Zookeeper,但是查询服务基于HTTP协议,仍然可以基于它们现有的数据为查询请求提供响应,即**Zookeeper停机也不影响Historical节点的现有数据可用性**.

   4. **Broker**节点

      - Historical节点和Real-time节点的查询路由.
      - Broker节点能够从Zookeeper中发布的元数据知道:哪些Segment可查询及其位置,将进来的查询请求路由到合适的Historical节点或者Real-time节点.
      - Broker节点先合并来自Historical节点和Real-time节点的结果片段成为一个最终结果,然后返回给请求者.
      - 缓存:LRU失效策略缓存（使用本地堆内存或者外部分布式key/value数据仓库，如Memcached）,实时数据不会缓存,总是被转发到Real-time节点
      - 可用性:如果Zookeeper停机,可以查询,但是Broker节点无法与之通信,Broker节点将会使用其最后一次获知的集群视图(view of the cluster),继续转发查询到Real-time节点和Historical节点

3. **Coordinator**节点

   - Historical节点上数据管理与分配:加载新数据,删除旧数据,复制数据,移动数据均衡负载
   - 使用多版本并发控制交换协议管理不可变数据块(Segment),以保持视图的稳定性.如果任何Segment包含了已被新的Segment完全废弃了的数据,旧的Segment就会从集群中删除
   - leader-selection选举出Coordinator,其余作为备份
   - 与Zookeeper连接获取集群当前信息
   - 与Mysql中表连接保存额外操作参数和配置信息,**关键信息**:包含应该由Historical节点提供的Segment列表的数据库表,任何可创建Segment的服务都可更新该表,比如Real-time节点
   - Mysql中还有一个规则表,管理:
     1. Segment如何被加载和删除信息
     2. 如何分层,以及多副本信息
     3. 何时从集群中完全销毁
   - 负载均衡
     - Segment分布式存储
     - 时间上相近的大Segment分发到不同的Historical节点
   - 复制:不同Historical节点可以复制同一Segment,每一层,复制数量完全可配置,设置高水平的容错能力,就可以设置相应多的副本数.被复制出来的Segment副本会被当作原节点同等对待,要遵循相同的负载分配算法.通过复制Segment,单个Historical节点故障在Druid集群中是透明的,这一特性可以无缝地将一个Historical节点摘除下线,升级.
   - 可用性:外部依赖与Zookeeper和Mysql
     - 依赖于Zookeeper确定哪些Historical节点已经在集群中存在,如果Zookeeper变得不可用,Coordinator将不能再发送指令来分配,均衡或删除Segment,但是不影响数据可用性
     - Mysql存储操作管理信息和应该存在于集群中的Segment的元数据信息,如果Mysql宕机,这些信息不可用,但是Broker,Historical和Real-time节点仍然可供查询,只是停止分配新的或删除旧的Segment

4. **存储格式**

   - Druid中的数据表是一系列**时间戳标记**的事件和被分片的一组Segment的集合,在数据源中每一个Segment通常包含500-1000万行数据.正式地,我们定义一个Segment为跨越一段时间范围内的一批数据行.Segment代表了Druid中的基本存储单元,并且数据的复制和分配都是在Segment层级上完成的.

   - Druid中**timestamp**列作为必选列,并且作为分配策略,数据保留策略和第一级的修剪方法.Druid根据定义好的时间区间(小时/天)将数据分配,也可能进一步将其他列的值分割,以达到期望的Segment的大小.Segment分片的时间粒度是一个数据量和时间范围的函数.

   - 数据块Segment由**数据源标识符**,数据的**时间间隔**和随数据创建而增加的**版本号字符串**来唯一标识.版本号字符串标识了数据块的新旧程度,对于一定时间范围内的数据块Segment,读操作总是根据这一时间范围内的最近的版本标识符来访问数据.

   - Druid最好被用于聚合时间流(所有流入Druid的数据必须带有timestamp),聚合信息按列存储,查询时只扫描加载需要的数据.

   - Druid拥有多个列类型来表示各种数据格式,根据列类型,使用不同的压缩方法来减少一列在内存和硬盘上的存储开销,比如可以使用**字典编码**来避免直接存储字符串带来的昂贵开销.

   - 数据**过滤索引**:一般来说,维度列包含字符串,指标列包含数值,Druid为字符串列创建了额外的查询索引,以至于只有跟指定查询过滤器相关的那些数据行才会被扫描.同时通过形成倒排索引,在位图集合上执行布尔运算,从而压缩数据和加快过滤查询,例如,我们有一份数据表:

     ![Druid 示例数据](../images/Druid 示例数据.png)

     我们可以将page的信息存储在一个二进制数组中,数组的索引代表我们的行,如果在某一行里面看到一个特定page,对应数组索引就会被标记为1,Justin Bieber -> rows [0, 1]  -> [1, 1, 0, 0],而Ke$ha -> rows[2, 3] -> [0, 0, 1, 1],因此为了知道数据行里包含Justin Bieber或者Ke$ha,可以将两个数组进行OR运算.

   - **存储引擎**:
     
     - Druid的持久化组件允许不同的存储引擎接入
     - 默认使用基于内存映射的存储引擎,依赖操作系统来个内存内外的数据块Segment分页,仅有被加载到内存中的数据块Segment能被扫描,基于内存映射的存储引擎允许最近的数据块Segment保存在内存页中,从不被用到的数据块Segment会被置换出去.**缺点**在于如果查询需要比一个给定的节点存储能力更多的数据块Segment载入内存页的时候,查询性能将面临数据块Segment出入内存页的开销问题.

5. **查询API**

   - 接受POST请求作为查询,请求body为一个包含key-value对的JSON对象,键值对指定了不同的查询参数
   - 典型查询:
     - 数据源:Datasource
     - 数据粒度:granularity
     - 目标时间范围:intervals
     - 查询类型:queryType
     - 指标:metrics
     - 过滤条件:filter
   - 查询类型:
     - TimeSeries
     - TopN
     - GroupBy
     - Select
     - Search(0.9版本新增)

6. **查询性能**

