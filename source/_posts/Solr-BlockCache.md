---
layout: title
title: Solr BlockCache
date: 2021-04-19 15:40:47
tags: Solr
categories: [分布式系统, 分布式检索, Solr]
---

# 概述

Solr 中为了加速索引在 HDFS 上的读写，增加了缓存，相关代码均位于 org.apache.solr.store.blockcache 包中。

# 源码分析

本篇源码基于 lucene-solr-8.5.2。

## 初始化

初始化的过程位于 HdfsDirectoryFactory 的 create 方法中，启用 BlockCache 需要配置对应参数，可参考 [Running Solr on HDFS](https://solr.apache.org/guide/7_2/running-solr-on-hdfs.html)，其中 BlockCache 可配置为全局的 BlockCache，也可以在每个 SolrCore 中创建单独的 BlockCache。NRTCachingDirectory 也是用于加速索引读取的，其内部使用的是 RAMDirectory（内存中的 Directory 实现），本文不予展开分析。

初始化的过程主要包含三个部分：
- BlockCache
- BlockDirectoryCache
- BlockDirectory

{% note warning %}
这里补充一下概念：默认地，每个 BlockCache 拥有 1 个 bank，这个 bank 下会有 16384 个 block，每个 block 是 (8192 / 1024) = 8K，像这样被称为一个 slab，其大小为 (16384 * 8192) / 1024 / 1024 = 128M。
{% endnote %}

```java
protected Directory create(String path, LockFactory lockFactory, DirContext dirContext) throws IOException {
    assert params != null : "init must be called before create";
    log.info("creating directory factory for path {}", path);
    Configuration conf = getConf(new Path(path));
    
    // metrics 是通过静态内部类 MetricsHolder 的单例模式构造的对象，是全局唯一的
    if (metrics == null) {
      metrics = MetricsHolder.metrics;
    }

    // 启用 BlockCache
    boolean blockCacheEnabled = getConfig(BLOCKCACHE_ENABLED, true);
    // 如果启用，对于每个节点上的集合都会使用一个 HDFS BlockCache
    // 如果禁用，每个 SolrCore 都会创建自己私有的 HDFS BlockCache
    boolean blockCacheGlobal = getConfig(BLOCKCACHE_GLOBAL, true);
    // 启用读 BlockCache
    boolean blockCacheReadEnabled = getConfig(BLOCKCACHE_READ_ENABLED, true);
    
    final HdfsDirectory hdfsDir;

    final Directory dir;
    // 判断是否启用 BlockCache
    if (blockCacheEnabled && dirContext != DirContext.META_DATA) {
      // 每个缓存片的块数
      int numberOfBlocksPerBank = getConfig(NUMBEROFBLOCKSPERBANK, 16384);

      // 缓存大小，默认值为 8192 即 8K
      int blockSize = BlockDirectory.BLOCK_SIZE;

      // 每个 BlockCache 的切片数
      int bankCount = getConfig(BLOCKCACHE_SLAB_COUNT, 1);

      // 启用直接内存分配（堆外内存），如果为 false 则使用堆内存
      boolean directAllocation = getConfig(BLOCKCACHE_DIRECT_MEMORY_ALLOCATION, true);

      // 每个切片的大小
      int slabSize = numberOfBlocksPerBank * blockSize;
      log.info(
          "Number of slabs of block cache [{}] with direct memory allocation set to [{}]",
          bankCount, directAllocation);
      log.info(
          "Block cache target memory usage, slab size of [{}] will allocate [{}] slabs and use ~[{}] bytes",
          new Object[] {slabSize, bankCount,
              ((long) bankCount * (long) slabSize)});
      
      int bsBufferSize = params.getInt("solr.hdfs.blockcache.bufferstore.buffersize", blockSize);
      int bsBufferCount = params.getInt("solr.hdfs.blockcache.bufferstore.buffercount", 0); // this is actually total size
      
      BlockCache blockCache = getBlockDirectoryCache(numberOfBlocksPerBank,
          blockSize, bankCount, directAllocation, slabSize,
          bsBufferSize, bsBufferCount, blockCacheGlobal);
      
      Cache cache = new BlockDirectoryCache(blockCache, path, metrics, blockCacheGlobal);
      int readBufferSize = params.getInt("solr.hdfs.blockcache.read.buffersize", blockSize);
      hdfsDir = new HdfsDirectory(new Path(path), lockFactory, conf, readBufferSize);
      dir = new BlockDirectory(path, hdfsDir, cache, null, blockCacheReadEnabled, false, cacheMerges, cacheReadOnce);
    } else {
      hdfsDir = new HdfsDirectory(new Path(path), conf);
      dir = hdfsDir;
    }
    if (params.getBool(LOCALITYMETRICS_ENABLED, false)) {
      LocalityHolder.reporter.registerDirectory(hdfsDir);
    }

    // 默认使用 NRTCachingDirectory 以达到近实时搜索的目的
    boolean nrtCachingDirectory = getConfig(NRTCACHINGDIRECTORY_ENABLE, true);
    if (nrtCachingDirectory) {
      double nrtCacheMaxMergeSizeMB = getConfig(NRTCACHINGDIRECTORY_MAXMERGESIZEMB, 16);
      double nrtCacheMaxCacheMB = getConfig(NRTCACHINGDIRECTORY_MAXCACHEMB, 192);
      
      return new NRTCachingDirectory(dir, nrtCacheMaxMergeSizeMB, nrtCacheMaxCacheMB);
    }
    return dir;
}
```

### BlockCache

当配置全局的 BlockCache 时，下面的方法保证了 BlockCache 是全局唯一共享的，理论上这里我觉得可以用 volatile 关键字修饰 globalBlockCache，然后再加上一层判断 globalBlockCache 是否为 null 后使用 synchronized 关键字，应该可以稍微提升一点效率，也就是采用双重校验锁的单例设计模式，不过此方法作为初始化方法也不会频繁进入，最新版尚未改动。

```java
private BlockCache getBlockDirectoryCache(int numberOfBlocksPerBank, int blockSize, int bankCount,
      boolean directAllocation, int slabSize, int bufferSize, int bufferCount, boolean staticBlockCache) {
    // 未配置 solr.hdfs.blockcache.global 为 false，每个 SolrCore 都会新创建一个 BlockCache
    if (!staticBlockCache) {
      log.info("Creating new single instance HDFS BlockCache");
      return createBlockCache(numberOfBlocksPerBank, blockSize, bankCount, directAllocation, slabSize, bufferSize, bufferCount);
    }
    // 默认配置全局 BlockCache，不会创建新的 BlockCache，而是共享
    synchronized (HdfsDirectoryFactory.class) {

      if (globalBlockCache == null) {
        log.info("Creating new global HDFS BlockCache");
        globalBlockCache = createBlockCache(numberOfBlocksPerBank, blockSize, bankCount,
            directAllocation, slabSize, bufferSize, bufferCount);
      }
    }
    return globalBlockCache;
}
```

在创建 BlockCache 之前会首先初始化 BufferStore，同时计算出分配的总内存。默认 directAllocation 是配置为 true 即开启堆外内存的，所以当堆外内存过小时，可能会提示 OOM 相关报错，需要指定 MaxDirectMemorySize 参数进行配置或者也可关闭堆外内存的分配。

```java
private BlockCache createBlockCache(int numberOfBlocksPerBank, int blockSize,
      int bankCount, boolean directAllocation, int slabSize, int bufferSize,
      int bufferCount) {
    BufferStore.initNewBuffer(bufferSize, bufferCount, metrics);
    long totalMemory = (long) bankCount * (long) numberOfBlocksPerBank
        * (long) blockSize;
    
    BlockCache blockCache;
    try {
      blockCache = new BlockCache(metrics, directAllocation, totalMemory, slabSize, blockSize);
    } catch (OutOfMemoryError e) {
      throw new RuntimeException(
          "The max direct memory is likely too low.  Either increase it (by adding -XX:MaxDirectMemorySize=<size>g -XX:+UseLargePages to your containers startup args)"
              + " or disable direct allocation using solr.hdfs.blockcache.direct.memory.allocation=false in solrconfig.xml. If you are putting the block cache on the heap,"
              + " your java heap size might not be large enough."
              + " Failed allocating ~" + totalMemory / 1000000.0 + " MB.",
          e);
    }
    return blockCache;
}
```

在初始化 BufferStore 时，将 shardBuffercacheLost、shardBuffercacheAllocate 与 metric 中的对应信息绑定，这样在后续的监控指标中能够获取具体的数据。新创建的 BufferStore 中，会调用至 setupBuffers 方法设置缓冲区，这个缓冲区会创建一个 bufferSize 大小的字节数组阻塞队列。

{% note warning %}
BufferStore 实现了接口 Store，其定义了两个方法，分别是取出缓存的 takeBuffer 方法和放入缓存的 putBuffer 方法，当成功取出缓存时，会增加 shardBuffercacheAllocate，而放入缓存失败时，则会增加 shardBuffercacheLost，以更新监控指标信息。
{% endnote %}

```java
public synchronized static void initNewBuffer(int bufferSize, long totalAmount, Metrics metrics) {
    if (totalAmount == 0) {
      return;
    }
    BufferStore bufferStore = bufferStores.get(bufferSize);
    if (bufferStore == null) {
      long count = totalAmount / bufferSize;
      if (count > Integer.MAX_VALUE) {
        count = Integer.MAX_VALUE;
      }
      AtomicLong shardBuffercacheLost = new AtomicLong(0);
      AtomicLong shardBuffercacheAllocate = new AtomicLong(0);
      if (metrics != null) {
        shardBuffercacheLost = metrics.shardBuffercacheLost;
        shardBuffercacheAllocate = metrics.shardBuffercacheAllocate;
      }
      BufferStore store = new BufferStore(bufferSize, (int) count, shardBuffercacheAllocate, shardBuffercacheLost);
      bufferStores.put(bufferSize, store);
    }
}
```

继续来看 BlockCache 的构造过程，每个 bank 都会为其创建一个对应的 BlockLocks 和 lockCounters，用于在缓冲时，检查是否能够找到位置进行缓存。默认配置了堆外内存，此处会进行分配，最大实例数为 16384 - 1 = 16383，当内存不足以分配时，会引发上述的 OOM 报错并提示相关信息。

这里的 cache 是用的 Google 的 Caffeine 本地缓存框架，并加入了监听器，当监听到文件删除时，会释放相应的缓存文件。当然，在关闭 BlockDirectoryCache 时，也会调用 BlockCache 中的 release 方法释放待删除的缓存文件。

{% note warning %}
cache 中存放的是 BlockCacheKey 和 BlockCacheLocation 的对应关系，其中 BlockCacheKey 包含 BlockID、已缓存的文件数、索引文件目录，BlockCacheLocation 包含 BankID、Bank 内 Block 的 bit 位、最后一次进入的时间和访问次数等。
{% endnote %}

```java
public BlockCache(Metrics metrics, boolean directAllocation,
      long totalMemory, int slabSize, int blockSize) {
    this.metrics = metrics;
    numberOfBlocksPerBank = slabSize / blockSize;
    int numberOfBanks = (int) (totalMemory / slabSize);
    
    banks = new ByteBuffer[numberOfBanks];
    locks = new BlockLocks[numberOfBanks];
    lockCounters = new AtomicInteger[numberOfBanks];
    maxEntries = (numberOfBlocksPerBank * numberOfBanks) - 1;
    for (int i = 0; i < numberOfBanks; i++) {
      if (directAllocation) {
        banks[i] = ByteBuffer.allocateDirect(numberOfBlocksPerBank * blockSize);
      } else {
        banks[i] = ByteBuffer.allocate(numberOfBlocksPerBank * blockSize);
      }
      locks[i] = new BlockLocks(numberOfBlocksPerBank);
      lockCounters[i] = new AtomicInteger();
    }

    // 用于监听文件删除，并释放缓存资源
    RemovalListener<BlockCacheKey,BlockCacheLocation> listener = (blockCacheKey, blockCacheLocation, removalCause) -> releaseLocation(blockCacheKey, blockCacheLocation, removalCause);

    cache = Caffeine.newBuilder()
        .removalListener(listener)
        .maximumSize(maxEntries)
        .build();
    this.blockSize = blockSize;
}
```

### BlockDirectoryCache

这里同样用 Caffeine 初始化了 names，names 中保存的是 {% label primary @缓存文件名 + 已缓存的文件数 %} 对应关系。BlockDirectoryCache 是该包中接口 Cache 的实现，定义了 6 个方法。这里的 setOnRelease 方法会将待释放资存储到 OnRelease 的 CopyOnWriteArrayList 中。在上面定义的监听器监听到文件删除时，会调用 releaseLocation 释放文件资源，并最终通过传入的 BlockCacheKey 删除 keysToRelease 中对应的 key。keysToRelease 存储了待释放的 BlockCacheKey，实际上是通过 BlockCache 的 release 方法调用至 cache.invalidate(Object key) 释放资源。

{% note warning %}
CopyOnWriteArrayList 是写数组的拷贝，支持高效率并发且是线程安全的，读操作无锁的 ArrayList，其本质是所有可变操作都通过对底层数组进行一次新的复制来实现，适合读多写少的场景。
{% endnote %}

{% note info %}
- `delete` - 从缓存中删除指定文件
- `update` - 更新指定缓存文件的内容，如有必要会创建一个缓存实例
- `fetch` - 获取指定的缓存文件内容，如果能找到缓存内容则返回 true
- `size` - 已缓存的实例数
- `renameCacheFile` - 重命名缓存中的指定文件，允许在不使缓存无效（即缓存有效）的情况下移动文件
- `releaseResources` - 释放与缓存相关联的所有文件资源
{% endnote %}

```java
public BlockDirectoryCache(BlockCache blockCache, String path, Metrics metrics, boolean releaseBlocks) {
    this.blockCache = blockCache;
    this.path = path;
    this.metrics = metrics;

    // 最多缓存 50000 的文件数
    names = Caffeine.newBuilder().maximumSize(50000).build();
    
    if (releaseBlocks) {
      // Collections 提供了 newSetFromMap 来保证元素唯一性的 Map 实现，就是用一个 Set 来表示 Map，它持有这个 Map 的引用，并且保持 Map 的顺序、并发和性能特征
      keysToRelease = Collections.newSetFromMap(new ConcurrentHashMap<BlockCacheKey,Boolean>(1024, 0.75f, 512));
      blockCache.setOnRelease(new OnRelease() {
        
        @Override
        public void release(BlockCacheKey key) {
          keysToRelease.remove(key);
        }
      });
    }
}
```

### BlockDirectory

BlockDirectory 继承自抽象类 FilterDirectory，该抽象类将调用委托给另一个 Directory 实现，如 NRTCachingDirectory，它们之间可以进行协作。cacheMerges、cacheReadOnce 默认均为 false，当判断是否使用读写缓存时，会用到这两个变量值。blockCacheFileTypes 是 Set<String> 类型，当用户指定了缓存的文件类型时，只针对符合文件后缀名的进行缓存，默认是 null，也就是说缓存所有类型的文件。blockCacheReadEnabled 默认为 true 即开启读缓存，可通过配置参数改变值；而 blockCacheWriteEnabled 默认为 false 即关闭写缓存，并且不可通过配置改变值。

```java
public BlockDirectory(String dirName, Directory directory, Cache cache,
                        Set<String> blockCacheFileTypes, boolean blockCacheReadEnabled,
                        boolean blockCacheWriteEnabled, boolean cacheMerges, boolean cacheReadOnce) throws IOException {
    super(directory);
    this.cacheMerges = cacheMerges;
    this.cacheReadOnce = cacheReadOnce;
    this.dirName = dirName;
    blockSize = BLOCK_SIZE;
    this.cache = cache;
    // 检查是否指定了缓存的文件类型，如 fdt、fdx...
    if (blockCacheFileTypes == null || blockCacheFileTypes.isEmpty()) {
      this.blockCacheFileTypes = null;
    } else {
      this.blockCacheFileTypes = blockCacheFileTypes;
    }
    this.blockCacheReadEnabled = blockCacheReadEnabled;
    if (!blockCacheReadEnabled) {
      log.info("Block cache on read is disabled");
    }
    this.blockCacheWriteEnabled = blockCacheWriteEnabled;
    if (!blockCacheWriteEnabled) {
      log.info("Block cache on write is disabled");
    }
}
```

## 写流程

从 BlockDirectory 的 createOutput 方法开始，该方法会在上层调用，在目录中创建一个新的空文件，并返回一个 IndexOutput 实例，用于追加数据到此文件。

{% note warning %}
注意：因为在 BlockDirectory 的构造方法中 blockCacheWriteEnabled 默认是 false，所以此处的 useWriteCache(name, context) 只会返回 false（方法此处不展开，感兴趣可自行查看源码），并且由于该值不能通过参数配置，所以用户只能通过改动代码后重新编译打包以支持此功能。
{% endnote %}

```java
public IndexOutput createOutput(String name, IOContext context)
      throws IOException {
    final IndexOutput dest = super.createOutput(name, context);
    if (useWriteCache(name, context)) {
      return new CachedIndexOutput(this, dest, blockSize, name, cache, blockSize);
    }
    return dest;
}
```

CachedIndexOutput 继承自 ReusedBufferedIndexOutput，在该类的构造方法中会从 BufferStore 中取出缓存准备好。directory.getFileCacheLocation(name) 方法则是将目录与索引文件名拼好作为变量 location 的值，每个 location 都是唯一的。

{% note warning %}
Segment 文件由于索引频繁的小合并，所以会不断改变其值，在缓存文件时要注意。
{% endnote %}

```java
public CachedIndexOutput(BlockDirectory directory, IndexOutput dest,
      int blockSize, String name, Cache cache, int bufferSize) {
    super("dest=" + dest + " name=" + name, name, bufferSize);
    this.directory = directory;
    this.dest = dest;
    this.blockSize = blockSize;
    this.name = name;
    this.location = directory.getFileCacheLocation(name);
    this.cache = cache;
}
```

创建完 IndexOutput 是为了实际写入数据，于是便会继续调用 writeByte 方法写入，当下一个要写入的字节 bufferPosition 大于等于 bufferSize 即 1024 时调用 flushBufferToCache 方法将缓冲的字节写入缓存，该方法会调用至 writeInternal 方法，然后调整下一个写入的位置和长度等信息。

```java
public void writeByte(byte b) throws IOException {
    if (bufferPosition >= bufferSize) {
      flushBufferToCache();
    }
    if (getFilePointer() >= fileLength) {
      fileLength++;
    }
    buffer[bufferPosition++] = b;
    if (bufferPosition > bufferLength) {
      bufferLength = bufferPosition;
    }
}
```

获取缓存文件中的位置，写入。

```java
public void writeInternal(byte[] b, int offset, int length)
      throws IOException {
    long position = getBufferStart();
    while (length > 0) {
      int len = writeBlock(position, b, offset, length);
      position += len;
      length -= len;
      offset += len;
    }
  }
```

获取 Block 的编号、偏移量和要写入的长度信息，先写入文件，再复制到缓存中。

```java
private int writeBlock(long position, byte[] b, int offset, int length)
      throws IOException {
    // read whole block into cache and then provide needed data
    // 将整个块读入缓存，然后提供所需数据，只有当数据大于 8192 右移后才能分配到新的 blockId
    long blockId = BlockDirectory.getBlock(position);
    int blockOffset = (int) BlockDirectory.getPosition(position);
    int lengthToWriteInBlock = Math.min(length, blockSize - blockOffset);
    
    // write the file and copy into the cache
    // 写入文件，并复制到缓存中
    dest.writeBytes(b, offset, lengthToWriteInBlock);
    // location：索引文件目录 + 文件名
    cache.update(location, blockId, blockOffset, b, offset,
        lengthToWriteInBlock);
    
    return lengthToWriteInBlock;
}
```

names 中存放的是缓存的文件名 + 已缓存的文件数（该值是通过原子类变量 counter 递增存入的），构造一个 BlockCacheKey 对象后，调用 BlockCache 的 store 方法存入相应值，成功后将其添加至待释放资源对象的 keysToRelease 中。

```java
public void update(String name, long blockId, int blockOffset, byte[] buffer,
      int offset, int length) {
    Integer file = names.getIfPresent(name);
    if (file == null) {
      file = counter.incrementAndGet();
      names.put(name, file);
    }
    BlockCacheKey blockCacheKey = new BlockCacheKey();
    blockCacheKey.setPath(path);
    blockCacheKey.setBlock(blockId);
    blockCacheKey.setFile(file);
    if (blockCache.store(blockCacheKey, blockOffset, buffer, offset, length) && keysToRelease != null) {
      keysToRelease.add(blockCacheKey);
    }
}
```

该方法可能会返回 false，这意味着无法缓存该 Block，也可能是已经缓存了该 Block，所以 Block 当前可能是未更新的，写流程分析至此。

```java
public boolean store(BlockCacheKey blockCacheKey, int blockOffset,
      byte[] data, int offset, int length) {
    if (length + blockOffset > blockSize) {
      throw new RuntimeException("Buffer size exceeded, expecting max ["
          + blockSize + "] got length [" + length + "] with blockOffset ["
          + blockOffset + "]");
    }
    BlockCacheLocation location = cache.getIfPresent(blockCacheKey);
    if (location == null) {
      location = new BlockCacheLocation();
      // 当缓存已满（正常情况）时，两次并发写会导致其中一个失败，一个简单的解决办法是留一个空的 Block，社区当前未做
      if (!findEmptyLocation(location)) {
        metrics.blockCacheStoreFail.incrementAndGet();
        return false;
      }
    } else {
      // 没有其他指标需要存储，不将冗余存储视为存储失败
      return false;
    }

    int bankId = location.getBankId();
    int bankOffset = location.getBlock() * blockSize;
    ByteBuffer bank = getBank(bankId);
    bank.position(bankOffset + blockOffset);
    bank.put(data, offset, length);

    cache.put(blockCacheKey.clone(), location);
    metrics.blockCacheSize.incrementAndGet();
    return true;
}
```

## 读流程

从 BlockDirectory 的 openInput 方法开始，该方法会在上层调用，创建一个 IndexInput 读取已有文件，符合条件则创建 CachedIndexInput，该类继承自抽象类 CustomBufferedIndexInput。

```java
private IndexInput openInput(String name, int bufferSize, IOContext context)
      throws IOException {
    final IndexInput source = super.openInput(name, context);
    if (useReadCache(name, context)) {
      return new CachedIndexInput(source, blockSize, name,
          getFileCacheName(name), cache, bufferSize);
    }
    return source;
}
```

而开始读取索引文件时，无非是几个方法，readByte 和 readBytes，都会调用一个比较重要的方法 refill，当没有数据时，会从 BufferStore 中取出缓存，获取相应的位置，调用 fetchBlock 方法，该方法会试着读取缓存文件内容，如果可以就直接返回，如果不可以则将文件读取至缓存或者更新缓存内容。

```java
private void refill() throws IOException {
    long start = bufferStart + bufferPosition;
    long end = start + bufferSize;
    if (end > length()) // don't read past EOF
    end = length();
    int newLength = (int) (end - start);
    if (newLength <= 0) throw new EOFException("read past EOF: " + this);
    
    if (buffer == null) {
      buffer = store.takeBuffer(bufferSize);
      seekInternal(bufferStart);
    }
    readInternal(buffer, 0, newLength);
    bufferLength = newLength;
    bufferStart = start;
    bufferPosition = 0;
}
```

在 fetchBlock 中，调用 checkCache 方法，然后调用至 BlockDirectoryCache 的 fetch 方法获取指定的缓存文件内容，如果能找到返回 true。

```java
public boolean fetch(String name, long blockId, int blockOffset, byte[] b,
      int off, int lengthToReadInBlock) {
    Integer file = names.getIfPresent(name);
    if (file == null) {
      return false;
    }
    BlockCacheKey blockCacheKey = new BlockCacheKey();
    blockCacheKey.setPath(path);
    blockCacheKey.setBlock(blockId);
    blockCacheKey.setFile(file);
    boolean fetch = blockCache.fetch(blockCacheKey, b, blockOffset, off,
        lengthToReadInBlock);
    return fetch;
}
```

直接获取缓存文件内容，如果没找到或者失效了则返回 false。

```java
public boolean fetch(BlockCacheKey blockCacheKey, byte[] buffer,
      int blockOffset, int off, int length) {
    BlockCacheLocation location = cache.getIfPresent(blockCacheKey);
    if (location == null) {
      metrics.blockCacheMiss.incrementAndGet();
      return false;
    }

    int bankId = location.getBankId();
    int bankOffset = location.getBlock() * blockSize;
    location.touch();
    ByteBuffer bank = getBank(bankId);
    bank.position(bankOffset + blockOffset);
    bank.get(buffer, off, length);

    if (location.isRemoved()) {
      // 必须在读取完成后检查，因为在读取之前或读取期间可能已将 bank 重新用于另一个块
      metrics.blockCacheMiss.incrementAndGet();
      return false;
    }

    metrics.blockCacheHit.incrementAndGet();
    return true;
}
```

未获取到指定的缓存文件内容，从文件系统中读取文件内容并加载至缓存，此处调用的 update 方法在写流程中已经分析过，该方法更新指定缓存文件的内容，如有必要也会创建一个缓存实例，以便下次读取，读流程分析至此。

```java
private void readIntoCacheAndResult(long blockId, int blockOffset,
                                        byte[] b, int off, int lengthToReadInBlock) throws IOException {
      long position = getRealPosition(blockId, 0);
      int length = (int) Math.min(blockSize, fileLength - position);
      source.seek(position);

      byte[] buf = store.takeBuffer(blockSize);
      source.readBytes(buf, 0, length);
      System.arraycopy(buf, blockOffset, b, off, lengthToReadInBlock);
      cache.update(cacheName, blockId, 0, buf, 0, blockSize);
      store.putBuffer(buf);
}
```