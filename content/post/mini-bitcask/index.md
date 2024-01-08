---
title: 基于 bitcask 存储模型的 mini kv 数据库实现
slug: mini-bitcask
date: 2023-10-23 00:00:00+0000
categories:
    - mini 系列
tags:
    - Rust
---

> 本文是 mini 系列的第四篇文章，mini 系列是一些知名框架或技术的最小化实现，本系列文章仅作为学习积累的目的，代码或架构在很大程度上参考了现有的一些源码或博客。本文基于 bitcask 存储模型使用 Rust 语言实现了一个高性能的持久化 kv 数据库。
> 
> 代码实现地址：https://github.com/zjregee/mini/mini-bitcask
## 一、bitcask 存储模型

bitcask 是一种键值存储引擎的存储模型，旨在提供高性能和简单的数据持久化。它主要用于构建分布式存储系统，通常用于支持键值对数据库。bitcask 的核心设计思想是将数据以追加日志的方式存储在磁盘上。这个追加日志是按照写入顺序不断增长的，新的写入会追加到文件的末尾。每个写入都会生成一个唯一的文件句柄，包含了对应数据的键、值、时间戳等信息。为了加速键的查找过程，bitcask 使用了内存中的哈希表或其他高效的数据结构来维护键和对应数据文件的索引。这使得在内存中进行键的查找非常高效。随着写入操作的进行，bitcask 的数据文件会不断增长。为了避免文件过大和提高性能，bitcask 会定期执行 merge 操作。merge 操作会将多个小的数据文件合并成一个更大的文件，并删除重复的键。这个操作确保在不断写入的同时，维护着较小的数据文件，提高读取性能。

相比于基于 LSM-Tree 数据结构的键值存储引擎，bitcask 的设计相对简单，易于理解和实现，容易维护和部署。基于 LSM-Tree 数据结构的键值存储引擎的写入操作先写入内存结构，后台异步合并到磁盘，bitcask 使用追加日志的方式顺序写入，两者都具有较高的写入吞吐量。对于读取性能而言，基于 LSM-Tree 数据结构的键值存储引擎在读取数据时可能需要访问多个不同层次跨越内存和磁盘之间的结构，相比于 bitcask 的内存索引可能有更高的读取延迟。相比基于 LSM-Tree 数据结构的键值存储引擎，bitcask 通过定期合并操作来维护数据文件，这可能导致在执行合并时出现停顿，影响实时性。此外，基于 LSM-Tree 数据结构的键值存储引擎的有序的键值存储结构设计对于范围查询操作更为高效。总体来说，基于 LSM-Tree 数据结构的键值存储引擎适用于更多的应用场景，在分布式数据库和存储系统中有着广泛的应用。
## 二、持久化索引原理与实现
### 2.1 目录模型

每一个 bitcask 实例对应一个目录，任何时刻只有一个操作进程可以打开这个目录提供存储服务。该目录中有一个处于 active 状态的文件用于写入数据，当该文件达到大小阈值后，关闭这个文件并创建一个新的处于 active 状态的文件。一旦一个文件被关闭，就被归档并不会再次打开写入数据。

处于 active 状态的文件仅通过在文件末尾追加键值条目的方式写入。删除操作同样通过追加特定的键值条目数据实现，实际值只会在合并操作时删除。bitcask 在内存中维护一个索引，记录每个 key 对应的 value 所存储的文件以及在文件中的偏移地址。当处理读取操作时，通过内存索引只需要一次磁盘 IO 即可获取 key 对应的 value。

为了减少磁盘的占用，删除一些已经被删除操作删除的值。bitcask 通过合并操作迭代目录中所有归档文件，并生成一组新的只包含每个 key 最新 value 的文件。在完成压缩操作后，还会在每个数据文件旁边创建一个元数据文件，元数据文件中包含相应数据文件中 key 对应 value 的位置和大小，以便加速数据库的构建过程。

> 除此之外，bitcask 官方文档所提到的一些细节：
> 
> 1）bitcask 依赖操作系统的文件系统缓存来提高读取性能；
> 
> 2）bitcask 的最初目标不是成为最快的存储引擎，而是获得足够的速度以及代码、设计和文件格式的高质量和简单性。在初步测试中，bitcask 在许多场景下可以优于其他快速存储系统；
> 
> 3）bitcask 不执行任何数据压缩，因为这样做的成本和收益非常依赖应用程序；
> 
> 4）通过 10 倍 RAM 数据集的初步测试证明 bitcask 能够处理比 RAM 大得多的数据集而不会性能降级；
> 
> 5）由于 bitcask 中的数据文件和提交日志是相同的操作，在崩溃恢复中不需要 replay，可以实现快速恢复且不丢失数据；
### 2.2 键值条目实现

键值条目是每次数据操作追加在文件末尾的数据条目，其内存数据结构如下所示：

```Rust
pub struct Entry {
    pub valid: bool,
    pub crc32: u32,
    pub key_size: u32,
    pub value_size: u32,
    pub state: u16,
    pub time_stamp: u64,
    pub key: Vec<u8>,
    pub value: Vec<u8>,
}

impl Entry {
    pub fn new(key: Vec<u8>, value: Vec<u8>, t: u16, mark: u16) -> Entry {...}
    pub fn new_with_expire(key: Vec<u8>, value: Vec<u8>, ddl: u64, t: u16, mark: u16) -> Entry {...}
    pub fn decode_header(buf: Vec<u8>) -> Option<Entry> {...}
    pub fn encode(&self) -> Option<Vec<u8>> {...}
    pub fn get_type(&self) -> u16 {...}
    pub fn get_mark(&self) -> u16 {...}
    pub fn size(&self) -> u32 {...}
}
```

键值条目数据结构可以分为两部分，长度为 22 字节的 header 区域和 data 区域。header 区域记录了 4 字节的校验码、4 字节的 key 长度、4 字节的 value 长度、2 字节的状态码、8 字节的时间戳。data 区域记录了 key 和 value 的实际数据，具体大小视 key 和 value 的大小而定。2 字节的状态码分为两部分，其中的一个字节用于记录 type，另一个字节用于记录 mark，状态码可用于标识该键值条目的操作类型，例如对删除操作进行特别标注。

键值条目数据结构的主要方法包括创建一个内存数据结构，反序列化 header 区域，序列化整个键值条目，获取状态码中的 type 和 mark，以及获取键值条目的大小。
### 2.3 文件抽象实现

每个存储在 bitcask 所管理目录中的文件可以用以下内存数据结构表示：

```Rust
pub struct DBFile {
    pub id: u32,
    pub path: String,
    pub offset: u32,
    pub file: Option<File>
}

impl DBFile {
    pub fn new(path: String, file_id: u32) -> Option<DBFile> {...}
    pub fn read(&self, mut offset: u32) -> Option<entry::Entry> {...}
    pub fn write(&mut self, entry: entry::Entry) -> bool {...}
}
```

文件数据结构记录了文件的 id 和路径，offset 记录了文件的大小，file 用于打开文件。文件数据结构的主要方法包括根据文件 id 和路径打开文件，并提供读取一个键值条目和写入一个键值条目的接口。读取键值条目的接口可用于迭代使用，直至顺序读取完文件中的所有键值条目。
## 三、kv 数据库实现

建立在键值条目和文件的抽象之上，我们可以实现如下的 kv 数据库数据结构。其中 config 记录了 kv 数据库的配置信息，hash_index 是 kv 数据库的内存索引，expires 用来管理键值的过期情况，active_file 记录了当前处于 active 状态的文件，arch_files 记录了已被归档的文件集合。

```Rust
#[derive(Default)]
pub struct Config {
    pub dir_path: String,
    pub max_file_size: u32,
}

#[derive(Default)]
pub struct kv {
    pub config: config::Config,
    pub hash_index: hash::Hash,
    pub expires: HashMap<String, u64>,
    pub active_file: Option<db_file::DBFile>,
    pub arch_files: HashMap<u32, db_file::DBFile>,
}

impl kv {
    pub fn open(config: config::Config) -> Option<kv> {...}
    pub fn get(&mut self, key: String) -> Option<String> {...}
    pub fn set(&mut self, key: String, value: String) -> bool {..}
    pub fn set_with_expire(&mut self, key: String, value: String, ddl: u64) -> bool {...}
    pub fn delete(&mut self, key: String) -> bool {...}
    pub fn clear(&mut self) -> bool {...}
    pub fn close(&mut self) {...}
    fn build_index(&mut self) {...}
    fn build_entry(&mut self, entry: entry::Entry) {...}
    fn store_entry(&mut self, entry: entry::Entry) -> bool {...}
    fn check_expired(&self, key: &String) -> bool {...}
}
```

数据库数据结构提供了诸多方法用于数据库构建和实现 kv 操作。open 函数用于打开指定配置下的数据库实例，get 函数用于查询 key，set 函数用于插入 key，set_with_expire 函数用于在插入 key 时设置 key 的过期时间，delete 函数用于删除 key，clear 函数用于清空数据库，close 函数用于关闭数据库，在关闭数据库时需要刷新文件系统缓存，避免数据未落盘造成数据丢失，build_index 函数用于构建数据库，build_entry 函数用于在构建数据库时通过键值条目更新内存索引，store_entry 函数用于在处于 ative 状态的文件中写入新的键值条目，check_expired 函数用于检查 key 是否过期。
### 3.1 数据库构建实现

数据库的构建过程首先需要遍历数据目录的所有数据文件，并提取数据文件名中的文件 id 信息，根据提取出的文件 id 信息，即可筛选出归档文件集合和处于 active 状态的文件（因为追加日志不断增长，id 最大的文件即为处于 active 状态的文件，如果不存在该文件，则创建一个新的处于 ative 状态的文件）。然后再将数据目录中的数据文件按照 id 从小到大排列依次打开迭代数据文件中的键值条目，根据每个键值条目代表的操作类型更新内存索引。当迭代完所有的数据文件，即数据库构建完成，可对外提供 kv 服务。
### 3.2 kv 操作实现

**查询 key**

查询 key 的过程首先需要检查 key 是否过期，如果 key 没有过期，则直接从内存索引中获取查询结果，如果 key 已经过期，则在内存索引删除相应的 key 和 value 并更新过期记录情况。

**插入key**

插入 key 首先需要在内存索引中插入相应的 key 和 value，然后生成相应的持久化键值条目写入处于 active 状态的文件。

**插入 key 并设置过期时间**

插入 key 并设置过期时间与插入 key 类似，在内存索引中插入相应的 key 和 value，然后生成相应的持久化键值条目写入处于 active 状态的文件（这里的持久化键值条目与插入 key 不同，有不同的条目类型并会额外记录 key 的过期时间），并更新过期记录情况。

**其他操作**

删除操作与清空操作同样会生成一个持久化键值条目，通过条目类型标识操作类型。
## 四、异步网络接口实现

至此已经实现了一个能够支持常规 kv 操作的数据库，本节通过 axum 和 tokio 框架为其实现一个异步的网络接口，可以通过 RESTful 接口访问 kv 服务。axum 是一个基于 Tokio 运行时用于构建异步、可扩展、高性能 Web 服务的 Rust 框架。下面是通过 axum 框架路由 HTTP 请求，启动网络服务并异步处理各个请求的部分代码示例，并行模式采用了 channel 进行通信的方式，维护一个数据库示例，并通过带缓冲区的 channel 获取待处理的请求。

```Rust
async fn kv_get (
    Json(payload): Json<serde_json::Value>,
    Extension(state): Extension<mpsc::Sender<Message>>,
) -> Json<Value> {...}

async fn kv_set (
    Json(payload): Json<serde_json::Value>,
    Extension(state): Extension<mpsc::Sender<Message>>,
) -> Json<Value> {...}

async fn kv_set_with_expire (
    Json(payload): Json<serde_json::Value>,
    Extension(state): Extension<mpsc::Sender<Message>>,
) -> Json<Value> {...}

async fn kv_delete (
    Json(payload): Json<serde_json::Value>,
    Extension(state): Extension<mpsc::Sender<Message>>,
) -> Json<Value> {...}

async fn kv_clear (
    Extension(state): Extension<mpsc::Sender<Message>>,
) -> Json<Value> {...}

async fn kv_close (
    Extension(state): Extension<mpsc::Sender<Message>>,
) -> Json<Value> {...}

#[tokio::main]
async fn main() {
    ...
    tokio::spawn(async move {
        let app = Router::new()
            .route("/key/get", post(kv_get))
            .route("/key/set", post(kv_set))
            .route("/key/set_with_expire", post(kv_set_with_expire))
            .route("/key/delete", post(kv_delete))
            .route("/key/clear", post(kv_clear))
            .route("/close", post(kv_close))
            .layer(Extension(tx));
        let addr = SocketAddr::from(([127, 0, 0, 1], port));
        println!("listening on {}", addr);
        axum::Server::bind(&addr)
            .serve(app.into_make_service())
            .await
            .unwrap();
    });
    ...
}
```
## 参考资料

* https://riak.com/assets/bitcask-intro.pdf