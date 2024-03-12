---
title: groupcache 支持热点填充的分布式内存缓存设计
slug: groupcache
date: 2023-12-11 00:00:00+0000
categories:
    - 源码阅读系列
tags:
    - Go
---

groupcache 是 Golang 官方库给出的一个分布式缓存实现示例，旨在提供一个简单、有效的分布式缓存解决方案，用于替代许多场景下同样用于分布式内存缓存的 memcached 集群。
## 一、分布式缓存架构

groupcache 集群包括多个对等的分布式节点，并通过 HTTP 请求的方式实现连接。每个节点对外提供的 HTTP 服务不仅用于处理客户端的缓存查询请求，同时也用于处理对等节点之间的缓存查询请求。值得一提的是，对于外部的客户端缓存查询请求，groupcache 节点通过 URL 路径解析的方式处理请求交互，并通过 Protobuf 的编解码方式进行节点间信息的处理。

```Go
type httpGetter struct {
    baseURL   string
    transport func(context.Context) http.RoundTripper
}

func (h *httpGetter) Get(ctx context.Context, in *pb.GetRequest, out *pb.GetResponse) error { ... }

type HTTPPool struct {
    Context     func(*http.Request) context.Context
    Transport   func(context.Context) http.RoundTripper
    self        string
    opts        HTTPPoolOptions
    mu          sync.Mutex
    peers       *consistenthash.Map
    httpGetters map[string]*httpGetter
}

func (p *HTTPPool) ServeHTTP(w http.ResponseWriter, r *http.Request) { ... }
```

HTTPPool 是每个节点中的内存数据结构，存储着整个集群的拓扑信息。对等的分布式节点之间通过一致性哈希算法来确定每个节点所管理的键值范围。一致性哈希算法的使用不仅可以使得集群中的节点快速定位到键值所属的对应节点，通过一致性哈希算法中副本数的设置也可以尽可能保障键值的均匀分配。此外，HTTPPool 通过 ServerHTTP 方法实现 http.Handler 接口实现对外提供的 HTTP 服务，httpGetter 提供了对其他节点进行访问的 Get 方法封装。

```Go
type ProtoGetter interface {
    Get(ctx context.Context, in *pb.GetRequest, out *pb.GetResponse) error
}

type PeerPicker interface {
    PickPeer(key string) (ProtoGetter, bool)
}

var portPicker func(groupName string) PeerPicker
```

groupcache 会将 HTTPPool 注册至 portPicker，从而实现缓存命名空间，不同命名空间的缓存服务使用不同的 HTTPPool。
## 二、缓存填充设计

HTTPPool 构建起了整个 groupcache 的网络拓扑结构并提供了 HTTP 请求流动的支持。在 groupcache 中，除了 HTTPPool，每个分布式节点还会维护自己的内存缓存。
### 2.1 ByteView 与 Sink

```Go
type ByteView struct {
    b []byte
    s string
}

type Sink interface {
    SetString(s string) error
    SetBytes(v []byte) error
    SetProto(m proto.Message) error
    view() (ByteView, error)
}
```

在缓存设计中，groupcache 提供了 ByteView 和 Sink 封装。ByteView 是缓存中的值类型，通过 ByteView 来避免缓存中的值发生变化。同时，这也意味着 groupcache 的设计初衷在于提供不可变的缓存查询支持，不支持数据的增删改。这在官方的文档中被称为对版本化值的不支持，没有缓存过期时间，没有显式的缓存驱逐（只会在缓存区满时对缓存进行淘汰），无需 CAS 操作（因为没有修改过程中的潜在冲突）。

支持增删改的分布式缓存系统需要保持强一致性，而并非所有场景都需要支持对缓存的增删改。大部分的缓存场景只需支持对高频访问数据的维护，即使存在需要改动的情况也可以将不会发生改变的东西剥离引入缓存。因此 groupcache 的设计理念在现实场景中仍然有一定的参考和使用价值，但并非适用于所有场景。Sink 是对缓存查询行为的封装，缓存查询过程中会将对应 ByteView 的值更新在实现 Sink 接口的对象中。
### 2.2 内存缓存架构与查询流程

```Go
type Getter interface {
    Get(ctx context.Context, key string, dest Sink) error
}

type GetterFunc func(ctx context.Context, key string, dest Sink) error

func (f GetterFunc) Get(ctx context.Context, key string, dest Sink) error {
    return f(ctx, key, dest)
}
```

groupcache 以 lib 库的形式提供给用户，在部署时与用户进程进行集成，用户需要为 groupcache 提供实现 Getter 接口的缓存获取方法。在之前已经提到，groupcache 中的每个节点通过一致性哈希算法确定自己负责的键值范围，这可以理解为这部分键值属于节点的本职工作。那么节点如何获取这部分键值呢，答案就是用户提供的 Getter 接口实现。实现 Getter 接口的方法可以从 redis 或其他数据库中获取实际数据，甚至是另外一个 groupcache 集群。groupcache 整体的接口设计以 lib 库的模式进行，而不直接负责运行分布式缓存服务，提供了使用上的灵活性。

```Go
type flightGroup interface {
    Do(key string, fn func() (interface{}, error)) (interface{}, error)
}

type Group struct {
    name       string
    getattr    Gettter
    peers      PeerPicker
    cacheBytes int64
    mainCache  cache
    hotCache   cache
    loadGroup  flightGroup
}

func (g *Group) Get(ctx context.Context, key string, dest Sink) error { ... }
func (g *Group) getLocally(ctx context.Context, key string, dest Sink) (ByteView, error) { ... }
func (g *Group) getFromPeer(ctx context.Context, key string, dest Sink) (ByteView, error) { ... }
func (g *Group) lookupCache(key string) (ByteView, bool) { ... }
func (g *Group) populateCache(key string, value ByteView, cache *cache) { ... }
```

Group 是内存缓存的主要数据结构，其中包括了 mainCache 和 hotCache 两种缓存，两者都是基于 LRU 算法实现的内存缓存。cacheBytes 用于限制缓存的大小，两种缓存加起来的内存使用量不大于 cacheBytes，否则就会通过 populateCache 方法驱逐部分缓存。mainCache 和 hotCache 中缓存的键值不仅仅来源于 Getter 方法，也包括接收到不属于节点管理范围的键值查询请求时，通过 getFromPeer 方法从其他对等节点获取的键值数据。

Group 接收到键值查询请求后，完整的处理流程包括：查询 mainCache 和 hotCache 中是否包含键值数据，存在缓存数据时直接返回相关数据；当缓存数据不存在时，判断键值数据是否有该节点直接负责，若由该节点直接负责该数据，则通过 getLocally 方法调用 Getter 方法获取数据，并添加至 mainCache；若由其他节点负责该数据，则通过 getFromPeer 方法从其他节点获取数据，并通过随机概率将该数据添加至 hotCache（通过引入随机性，使得 hotCache 中缓存的是其他节点中的热点数据，即当需要经常从其他节点查询某数据时，则有较大概率将数据加入 hotCache）。
### 2.3 缓存未命中下的限流

至此，groupcache 的整体架构以及实现分布式缓存的原理已经基本清晰。groupcache 通过两种方式确保了缓存未命中下的限流：

* 所有对等节点均匀划分了键值范围，当缓存未命中时，只会有一个节点从后端的数据库中查询键值数据，其他节点只需从该节点同步数据，避免大量的重复查询流量直接引入后端的数据库中，这被成为缓存填充的协调以及键值加载后的多路复用。
* 通过 flightGroup 方法避免了缓存未命中并有大量查询请求在查询同一个未命中的键值时生成众多相同的缓存更新操作。这里有两层意思，缓存未命中时，节点只会向其他负责该数据的节点发送一个查询请求，负责该数据的节点只会向后端的数据库发送一个查询请求。

```Go
type call struct {
    wg sync.WaitGroup
    val interface{}
    err error
}

type Group struct {
    mu sync.Mutex
    m map[string]*call
}

func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
    g.mu.Lock()
    if g.m == nil {
        g.m = make(map[string]*call)
    }
    if c, ok := g.m[key]; ok {
        g.mu.Unlock()
        c.wg.Wait()
        return c.val, c.err
    }
    c := new(call)
    c.wg.Add(1)
    g.m[key] = c
    g.mu.Unlock()
    c.val, c.err = fn()
    c.wg.Done()
    g.mu.Lock()
    delete(g.m, key)
    g.mu.Unlock()
    return c.val, c.err
}
```

上面的代码是 groupcache 中对 flightGroup 的实现。

> 写在最后。
> 
> groupcache 整体代码在不包括测试的情况下一千多行，麻雀虽小，五脏俱全，其中包含的细节和工程实现还是比较完整的，对缓存穿透和热点节点问题给出了一定的解决思路。据官网所描述，groupcache 也曾在 Google 中用于生产环境，其设计思想并没有很复杂，但仍然具有一定参考价值。此外，由于灵活的运用了接口和 lib 库设计模式使得代码并没有非常直观，没有一个端到端的运行链，很多时候逻辑是循环嵌套的。
> 
> 项目地址：https://github.com/golang/groupcache.