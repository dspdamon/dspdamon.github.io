---
layout: post
title: 'HBase预分region未生效'
key: 20180905
category: Document
tags:
- HBase
- 大数据
---

本文介绍region数比预分的region个数多，出现大量split过后的新region
<!--more-->

# 问题现象

配置的某两张表A表和B表的预分区是45和60，但是查看发现实际的region数是100+不符合预期。

# 分析过程

1. 针对A数据表，通过检索hmaster的日志，发现这个表在数据插入过程中，region发生了split， 但是按照当前客户配置的region spit的阈值是10G，即只有region大小超过10G才会发生split，理论上不应该发生split。因为这张表的大小为20G左右，按照预分45个region算，平均每个region 只有455MB左右。
2. 首先查看hbase源码中判断一个region是否会发生split的逻辑(org.apache.hadoop.hbase.regionserver. IncreasingToUpperBoundRegionSplitPolicy)
```
@Override
  protected boolean shouldSplit() {
    boolean force = region.shouldForceSplit();
    boolean foundABigStore = false;
    // Get count of regions that have the same common table as this.region
    int tableRegionsCount = getCountOfCommonTableRegions();
    // Get size to check
    long sizeToCheck = getSizeToCheck(tableRegionsCount);

    for (Store store : region.getStores()) {
      // If any of the stores is unable to split (eg they contain reference files)
      // then don't split
      if (!store.canSplit()) {
        return false;
      }

      // Mark if any store is big enough
      long size = store.getSize();
      if (size > sizeToCheck) {
        LOG.debug("ShouldSplit because " + store.getColumnFamilyName() + " size=" + size
                  + ", sizeToCheck=" + sizeToCheck + ", regionsWithCommonTable="
                  + tableRegionsCount);
        foundABigStore = true;
      }
    }

    return foundABigStore | force;
  }
```
   可以看到当region中任一一个store的数据大小超过sizeToCheck就会进行split。以A表为例，store.size大约为 totalTableSize/preRegionCount = 20G/45 = 455MB，

3. 接下来查看一下sizeTocheck如何计算的
   
```
protected long getSizeToCheck(final int tableRegionsCount) {
    // safety check for 100 to avoid numerical overflow in extreme cases
    return tableRegionsCount == 0 || tableRegionsCount > 100
               ? getDesiredMaxFileSize()
               : Math.min(getDesiredMaxFileSize(),
                          initialSize * tableRegionsCount * tableRegionsCount * tableRegionsCount);
  }
```

如上，针对A这张表来说，初始化预分区45个region， 而当前集群大小为50个节点regionserver的集群上，平均每个regionServer不到1个region, 即只有45个regionserver会平均到1个region。因此，tableRegionCount=1.
通过查看配置可知：
initSize为256MB（hbase.increasing.policy.initial.size）
getDesireMaxFileSize() 为10GB， 配置为"hbase.hregion.max.filesize"
则sizeToCheck = min(10GB, 256MB*1*1*1) = 256MB.
所以，只要这个表的初始化region大小超过了256MB，就会发生一次split。

# 优化建议
1. 建议将每个表的预分区调整为region server的节点数，以便region更加均匀分布，提升读写性能。

2. 建议将hbase.increasing.policy.initial.size调整大于90%数据表的平均预分的region大小，以便大部分的数据表region能够按照预期的分区数增长