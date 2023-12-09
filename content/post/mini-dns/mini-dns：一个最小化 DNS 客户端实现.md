---
title: mini-dns：一个最小化 DNS 客户端实现
date: 2023-09-06 00:00:00+0000
categories:
    - mini 系列
tags:
    - Rust
---

## 一、DNS 域名解析流程
一个完整的DNS 域名解析流程如下：
* 本地电脑检查浏览器缓存中是否有域名对应的解析过的 IP 地址，如果缓存中有，则结束解析过程。浏览器缓存域名是有限制的，不仅浏览器缓存大小有限制，而且缓存的时间也有限制，通常情况下为几分钟到几小时不等，域名被缓存的时间限制可以通过 TTL 属性来设置。
* 如果浏览器缓存中没有数据，浏览器会查找操作系统缓存中是否有这个域名对应的 DNS 解析结果。在 Linux 中可以通过 /etc/hosts 文件来将任何域名解析到任何能够访问的 IP 地址。
* 当前两个过程无法解析时，就要用到我们网络配置中的 DNS 服务器地址了。操作系统会将这个域名发送给本地 DNS 服务器，后续的 DNS 迭代和递归也是由本地 DNS 服务器负责。
## 二、DNS 协议介绍

### 2.1 DNS 记录类型
DNS 记录是存储在域名系统数据库中的数据项，用于将域名与其他信息关联起来。这些记录包含了关于域名的各种信息，例如与域名相关联的 IP 地址、邮件服务器信息、文本信息等。DNS 记录有多种类型，每种类型有不同的用途。下面是一些常见的 DNS 记录类型：
* A 记录：映射域名到一个 IPv4 地址；
* AAAA 记录：映射域名到一个 IPv6 地址；
* CNAME 记录：创建域名的别名，将一个域名指向另一个域名；
* MX 记录：指定接收域的电子邮件的邮件服务器；
* PTR 记录：用于将 IP 地址映射回域名，通常用于反向 DNS 查找；
* NS 记录：指定域名的权威域名服务器，负责存储该域的 DNS 记录；
* TXT 记录：存储与域名相关的文本信息，通常用于验证域名所有权或提供其他信息；
* SOA 记录：包含有关域的权威信息，如域的主要域名服务器、域的管理员和域的参数；
* SRV 记录：用于指定特定服务的主机和端口。
这些记录协同工作，搭建了一个完善的层次结构。
### 2.2 DNS 报文格式
DNS 报文通常采用 UDP 进行传输，报文长度限制为 512 字节。DNS 使用相同的查询和响应格式，DNS 数据包的格式如下：
* Header：12 字节，查询和响应的相关信息；
* Question Section：大小可变，表明查询名称和感兴趣的记录类型；
* Answer Section：大小可变，所请求类型的相关记录；
* Authority Section：大小可变，名称服务器列表，用于递归地解析查询；
* Additional Section：大小可变，可能有用的一些记录，例如记录 NS 记录对应的 A 记录。

Header 的结构如下：

| RFC Name | Descriptive Name | Length | Description |
| --- | --- | --- | --- |
| ID | Packet Identifier | 16 bits | 为查询报文分配一个随机标识，响应报文必须使用相同的标识进行应答。这是由于 UDP 的无状态特性，区分响应所必须的。 |
| QR | Query Response | 1 bit | 0 代表查询报文，1 代表响应报文。 |
| OPCODE | Operation Code | 4 bits | 通常为 0，具体的细节要参考 RFC 1035。 |
| AA | Authoritative Answer | 1 bit | 如果响应服务器是权威服务器（即它拥有所查询的域），则设置为 1。 |
| TC | Truncated Message | 1 bit | 如果报文长度超过 512 字节，则设置为 1。传统上用于提示使用 TCP 重新发出查询，TCP 不限制长度。 |
| RD | Recursion Desired | 1 bit | 由请求的发送方设置，如果服务器在没有现成的答案时需要尝试递归地解析查询。 |
| RA | Recursion Available | 1 bit | 由服务器设置，指示是否允许递归查询。 |
| Z | Reserved | 3 bits | 最初作为保留字段，现在用于 DNSSEC 查询。 |
| RCODE | Response Code | 4 bits | 由服务器设置，指示响应是否成功，在失败时提供有关失败原因的详细信息。 |
| QDCOUNT | Question Count | 16 bits | Question Section 中的记录数。 |
| ANCOUNT | Answer Count | 16 bits | Answer Section 中的记录数。 |
| NSCOUNT | Authority Count | 16 bits | Authority Section 中的记录数。 |
| ARCOUNT | Additional Count | 16 bits | Additional Section 中的记录数。 |
单个 Question Section的记录包含如下字段：
* Name：被编码成一个标签序列的域名，会在之后进一步说明；
* Type：2 字节的整型，代表记录的类型；
* Class：2 字节的整型，在实践中通常为 1。

单个 Answer Section 的记录包含如下字段：
* Name：被编码成一个标签序列的域名；
* Type：2字节的整型，代表记录的类型；
* Class：2 字节的整型，在实践中通常为 1；
* TTL：4 字节的整形，记录可以缓存的时间；
* Len：2 字节的整型，记录类型特定数据的长度；
* Data：记录的数据，例如当记录为 A 类型时，Data 中的数据是 4 字节整型编码的 IP 地址。

### 2.3 DNS 操作示例
在这一部分会使用 dig 命令行工具来直观的感受 DNS 协议的实际使用。dig 是一个用于查询 DNS 信息的网络工具，可以用于诊断网络问题，验证 DNS 配置以及获取与域名相关的信息。
![[截屏2023-11-23 16.34.17.png]]
我们可以看到 dig 明确描述了响应报文的 header、question section 和 answer section。header 中的 opcode 使用 OPCODE QUERY，对应于 0。status 也就是 RCODE，被设置为 NOERROR，对应于 0。id 为 4260，在重复查询时会随机更改。启用了 QR、RD、RA 标志位，它们的数值为 1。最后 header 还告诉了我们在 query section 和 answer section 分别有一个记录。answer section 显示了查询的结果，其中 IN 表示类，137 是 TTL，A 告诉我们我们在查询 A 记录，以及 google.com 的 IP 为 8.7.198.46。最后我们还知道了报文的总大小为 44 字节。
### 2.4 一些更复杂的现实

## 三、报文序列与反序列化实现

## 四、DNS 客户端实现

## 五、本地客户端部署与实践
在之前介绍 DNS 域名解析流程的时候我们曾提到过，电脑中的 DNS 缓存分为两部分，一个是由操作系统管理的系统缓存，一个是由浏览器管理的 DNS 缓存。这两个缓存通常都是存储在内存里，用于提高域名解析的速度，而没有保存在磁盘上的文件系统中。
操作系统提供的系统 DNS 缓存以及 DNS 解析能力根据操作系统的不同而存在一定差异。以 Ubuntu 为例，Ubuntu使用 systemd-resolved 负责处理域名解析和 DNS 服务。systemd-resolved 是 systemd 体系结构中的一部分，它提供了一个本地的 DNS 解析器和缓存。systemd-resolved 在本地维护一个 DNS 缓存，用于存储之前解析的域名信息，这有助于避免重复向 DNS 服务器发出相同的查询请求，提高了解析的效率。systemd-resolved 还提供了 DNSSEC 支持，用于验证 DNS 响应的真实性和完整性，以提高安全性，以及 mDNS 支持，这是一种用于本地网络上的无配置服务发现的多播 DNS 协议。浏览器所需要的 DNS 解析能力也是建立在使用 systemd-resolved 提供的服务的基础上。
我们之前所实现的 DNS 客户端并不是为了替代 systemd-resolved，而是我们可以将我们所实现的 DNS 客户端与 systemd-resolved 协同工作。再加入 DNS 客户端后一个完整的 DNS 解析流程如下：
* 操作系统或浏览器需要 DNS 服务时，首先将查询请求发送给 systemd-resolved；
* systemd-resolved 查看本地缓存，如果没有缓存命中，则先去查询 hosts 文件。如果仍没有相关信息，则将查询请求转发至本地 DNS 客户端。
* 本地 DNS 客户端完成后续的查询请求，并将结果返回给 system-resolved。

协同配置 DNS 客户端与 systemd-resolved 的方法如下：
* 运行本地 DNS 客户端在某一端口，监听 DNS 查询请求；
* 修改 systemd-resolved 配置文件 /etc/systemd/resolved.conf，指向本地客户端监听的地址和端口。

这样做的好处主要有两点。首先通过我们实现的 DNS 客户端，可以更灵活地控制本地域名解析，可以配置一些特殊域名或区域的处理，实现本地定制的 DNS 解析。其次，本地 DNS 客户端可以通过持久化方式缓存一些更长期的域名对应关系，但客观来说，协同工作对于性能提升还是有限的，因为，在我们实际的网络环境中，即使需要通过网络连接来发送 DNS 查询请求也不会跑多远就遇到诸多缓存，很难看到有明显的性能提升。
在完成部署后，我们继续用 dig 工具来测试本地 DNS 服务器。