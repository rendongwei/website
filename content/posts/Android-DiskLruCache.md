---
title: "Android-DiskLruCache"
date: "2022-11-07"
categories:
    - "Android"
tags:
    - "Android"
    - "Android框架"
toc: true
indent: false
original: true
draft: false
---

### 什么是DiskLruCache

是文件缓存的管理对象，使用 LRU 算法对保存在永久存储设备上的缓存文件进行管理。

比手机的`闪存`更低速的访问设备是网络，文件缓存的意义就在于通过重复利用缓存的数据，减少网络请求构造方法是 private 修饰的，无法使用。，减少网络流量，提高响应速度。

### 用法

1. **使用静态方法 `static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)` 来创建 `DiskLruCache` 对象。**
   1. 构造方法是 `private` 修饰的，无法使用。
   2. 参数 `directory` 表示保存的目录，注意外部目录权限问题。
   3. 参数 `appVersion` 可以设置为版本号 version code。
   4. 参数 `valueCount` 表示一个 key 可以关联几个文件，一般为 1（一般情况关联多个没有必要，而且会增加编码复杂度）。
   5. 参数 `maxSize` 缓存大小限制，单位 Byte。
   6. 需要调用 `close()` 关闭。
2. **读取缓存**
   1. 先调用 `DiskLruCache.Snapshot get(String key)` 获取一个 `DiskLruCache.Snapshot` 对象，再通过这个对象进行读取操作。
   2. 返回的类型是 `DiskLruCache.Snapshot`，其实就是这个 `key` 对应的相关数据，主要是多个文件的输入流和大小，多个文件的数量对应构造方法里的 `valueCount`。
   3. `Snapshot.getLength(int index)` 获取文件大小，进行一些判断。
   4. `Snapshot.getInputStream(int index)` 获取一个 `InputStream`，可以用来读取文件内容，注意这个流不是缓存流，如果需要缓存流可以创建 `BufferedInputStream` 包装一下 `InputStream`。
   5. 这个 `InputStream` 需要手动关闭，既可以直接关闭 `InputStream`，也可以调用 `Snapshot.close()` 来关闭属于它的所有 `InputStream`。
3. **写入缓存**
   1. 先调用 `DiskLruCache.Editor edit(String key)` 方法获取一个 `DiskLruCache.Editor` 对象，再通过这个对象进行写入操作。
   2. 返回的类型是 `DiskLruCache.Editor`，其实就是将写入相关的一些操作抽象处理，对这个对象的操作都对应 `key` 关联的缓存文件。
   3. 如果同时有另一个 `Editor` 对象是通过 `key` 获取的，`edit` 方法将返回 null。保证同时只有一个 `Editor` 对象在对同一个 `key` 进行写入操作。因此调用之后需要判断一下。
   4. `OutputStream newOutputStream(int index)` 创建输出流来写入数据，注意这个流不是缓存流，如果需要缓存流可以创建 `BufferedOutputStream` 包装一下 `OutputStream`。
   5. 这个 `OutputStream` 需要手动关闭。
   6. 除了关闭输出流，还还需对 `Editor` 设置结果。如果写入操作和相关业务成功了，缓存文件有效，则调用 `Editor.commit()` 方法表示缓存写入成功。如果写入操作或相关业务失败了，缓存文件无效，则调用 `Editor.abort()` 来还原为未获取 `Editor` 之前的状态。

### 线程安全和一致性

+ `DiskLruCache` 管理多个 Entry（key-values），因此锁粒度应该是 Entry 级别的。

+ `get` 和 `edit` 方法都是同步方法，保证内部的 Entry Map 的安全访问，是保证线程安全的第一步。
+ `get` 和 `edit` 方法都返回一个对象来关联某个 Entry
  + 对读取来说，允许多个对象同时读取，不需要加锁
  + 对写入来说，`edit` 方法内部保证不会有两个 `Editor` 同时关联一个 Entry。直接利用方法本身的锁就达到了目的。
+ 可以同时读写一个 Entry，读和写不互相影响，读的是快照，写是原子操作。
  + 读取方法 `get` 返回后，就像返回值类型 `Snapshot` 的词义暗示的那样，它是一个快照对象，再对这个 Entry 进行任何操作都不会影响快照对象，快照对象在返回的时候就固定了，关联的输入流也一样。
  + 完成一次写入必须调用 `commit` 方法，`commit` 方法是原子操作，多个 value 的修改要么同时不可见，要么同时可见。

### 注意

+ 缓存空间的大小限制并不是特别精确，体现在并不一定及时删除应该删除的缓存文件，大小的计算也不包括内部文件以及文件系统的额外消耗。对空间敏感的使用者应该设置一个保守的大小。
+ open 方法的 appVersion 其实不必是应用的 version code。因为 appVersion 属性的主要作用是在升级app后清空缓存文件。DiskLruCache 这样做的原因是假定 app 的升级会导致缓存数据与新代码不兼容，可以说这是一种保守的策略。如果你能分辨出是否有不兼容问题，那么就可以随意定制 appVersion 这个参数，减少不必要的全局删除。如果不可能出现不兼容问题，那么就直接设置为一个固定值就可以了。
+ key 必须 match 正则表达式 `[a-z0-9_-]{1,64}`，在实际使用中，可以用 md5 的方法将 url 转换为符合条件的字符串。
+ 缓存目录必须只能用于 `DiskLruCache` 缓存，不能再用作别的目的，因为 `DiskLruCache` 可能会删除或覆盖其中的文件。
+ 不要多进程使用同一个缓存目录，可能会发生错误。

### 实现原理

DiskLruCache 内部使用一个 journal 文件来记录和管理缓存文件。文件内容大概长这样：

```
libcore.io.DiskLruCache
1
100
2

CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
DIRTY 335c4c6028171cfddfbaae1a9c313c52
CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
REMOVE 335c4c6028171cfddfbaae1a9c313c52
DIRTY 1ab96a171faeeee38496d8b330771a7a
CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
READ 335c4c6028171cfddfbaae1a9c313c52
READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
```

journal 文件记录了一些元信息，比如版本号什么的。

journal 文件还记录了每个 Entry 的数据，有四种状态：

+ DIRTY：表示正在写入
+ CLEAN：表示就绪，可以读取到最新的修改了
+ REMOVE：表示被删除了
+ READ：表示正在读取

Entry 数据并不是在记录的位置“原地”修改，而是不停地添加新的状态到文件末尾，只有读取到最新的一条 Entry 相关的记录才能知道它的最新状态。随着操作越来越多，DiskLruCache 也会执行压缩，删除之前的状态。

