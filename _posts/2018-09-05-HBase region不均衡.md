---
layout: post
title: 'HBase region个数不均衡'
key: 20180905
category: Document
tags:
- HBase
- 大数据
---

本文介绍regionserver节点中region个数不均衡的调整过程。
<!--more-->

# 问题现象

Hbase系统运行一段时间后，发现每个regionserver节点上的region数目不均匀，有的节点的region数量相差有2-3倍。

# 分析过程
1. 查看HDFS数据分布情况，每个节点的数据都是均匀的，排除由于HDFS的数据不均导致。
2. 分析是否后台balance没有生效，查看日志分析，可知后台balancer功能正常运行，每5min执行一次做balance，但是当前集群状态并不会再在进行region 迁移
3. 查看配置的balance算法为StochasticLoadBalancer（org.apache.hadoop.hbase.master.balancer. StochasticLoadBalancer）
```
protected boolean needsBalance(Cluster c) {
    ClusterLoadState cs = new ClusterLoadState(c.clusterState);
    if (cs.getNumServers() < MIN_SERVER_BALANCE) {
      if (LOG.isDebugEnabled()) {
        LOG.debug("Not running balancer because only " + cs.getNumServers()
            + " active regionserver(s)");
      }
      return false;
    }
    if(areSomeRegionReplicasColocated(c)) return true;
    // Check if we even need to do any load balancing
    // HBASE-3681 check sloppiness first
    float average = cs.getLoadAverage(); // for logging
    int floor = (int) Math.floor(average * (1 - slop));
    int ceiling = (int) Math.ceil(average * (1 + slop));
    if (!(cs.getMaxLoad() > ceiling || cs.getMinLoad() < floor)) {
      NavigableMap<ServerAndLoad, List<HRegionInfo>> serversByLoad = cs.getServersByLoad();
      if (LOG.isTraceEnabled()) {
        // If nothing to balance, then don't say anything unless trace-level logging.
        LOG.trace("Skipping load balancing because balanced cluster; " +
          "servers=" + cs.getNumServers() +
          " regions=" + cs.getNumRegions() + " average=" + average +
          " mostloaded=" + serversByLoad.lastKey().getLoad() +
          " leastloaded=" + serversByLoad.firstKey().getLoad());
      }
      return false;
    }
    return true;
  }
```

    可以看出如果load在floor和ceilling范围内则不需要balance
4. 当前集群计算如下，集群RegionServer 个数: 55
    Total Region数: 38460
    每个regionserver平均regio数：699
    每个regionserver最大值：838
    每个regionserver最小值：559

    因此，平均每个regionserver的region数范围应该在559 ~ 838之间。
    而从当前集群的region 分布情况看，最大的900+， 最小的300+， 推断还有其他的因素影响了region分布

5. 继续分析StochasticLoadBalancer的均衡策如下：

    ```
    regionLoadFunctions = new CostFromRegionLoadFunction[] {
          new ReadRequestCostFunction(conf),
          new WriteRequestCostFunction(conf),
          new MemstoreSizeCostFunction(conf),
          new StoreFileCostFunction(conf)
        };
    
        regionReplicaHostCostFunction = new RegionReplicaHostCostFunction(conf);
        regionReplicaRackCostFunction = new RegionReplicaRackCostFunction(conf);
    
        costFunctions = new CostFunction[]{
          new RegionCountSkewCostFunction(conf),
          new PrimaryRegionCountSkewCostFunction(conf),
          new MoveCostFunction(conf),
          localityCost,
          new TableSkewCostFunction(conf),
          regionReplicaHostCostFunction,
          regionReplicaRackCostFunction,
          regionLoadFunctions[0],
          regionLoadFunctions[1],
          regionLoadFunctions[2],
          regionLoadFunctions[3],
        };
    ```
    
   StochasticLoadBalancer 这种策略真的是非常复杂，简单来讲，是一种综合权衡一下6个因素的均衡策略：
   - 每台服务器读请求数(ReadRequestCostFunction)
   - 每台服务器写请求数(WriteRequestCostFunction)
   - Region个数(RegionCountSkewCostFunction)
   - 移动代价(MoveCostFunction)
   - 数据locality(TableSkewCostFunction)
   - 每张表占据RegionServer中region个数上限(LocalityCostFunction)
   >  https://www.cnblogs.com/xjh713/p/6909076.html?utm_source=itdadao&utm_medium=referral


6. 从源码分析中可以发现有个配置参数hbase.master.loadbalance.bytable，对region数量的分布起着关键作用

    ```
     * This balancer is best used with hbase.master.loadbalance.bytable set to false
 * so that the balancer gets the full picture of all loads on the cluster.
    ```
    而集群配置中该参数配置为true，修改为false后region个数平均
    
# 总结

1. 如果将hbase.master.loadbalance.bytable配置为false, 则意味着hbase会从整个集群的维度来考虑region的均衡，最大可能到保证了每个regionserver上的region数是均衡的。保证了不会存在hbase服务重启后，因为个别regionserver的region数过大，而导致region无法上线的可靠性问题发生。

2. 而在hbase在做banance的时候，不会考虑table级别的均衡。即有可能会导致某张表的多个热点region都分布在了一个regionserver上。在这种情况下，则需要用户从业务识别，手动将该表的热点region迁移到多个regionserver，均衡压力。

    
    