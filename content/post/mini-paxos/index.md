---
title: 分布式共识算法之 Paxos 算法模拟
slug: mini-paxos
date: 2023-10-07 00:00:00+0000
categories:
    - mini 系列
tags:
    - Rust
---

> 本文是 mini 系列的第三篇文章，mini 系列是一些知名框架或技术的最小化实现，本系列文章仅作为学习积累的目的，代码或架构在很大程度上参考了现有的一些源码或博客。本文通过 C++ 语言模拟了 Paxos 算法中核心的二阶段提议提交。
> 
> 代码实现地址：https://github.com/zjregee/mini/mini-paxos
## 一、分布式共识算法

分布式共识算法是一种用于在分布式系统中达成一致性的方法。在分布式系统中，多个计算机节点之间需要协作完成任务，但由于网络延迟、节点故障等问题，节点之间可能产生不一致的状态。共识算法的目标是确保所有节点就系统的状态达成一致，即使在存在故障或网络分区的情况下也能保持一致。常见的分布式共识算法包括 Paxos 算法和 Raft 算法等，其中每个算法的特点如下：

* Paxos 算法是一种经典的分布式共识算法，用于解决一致性问题。它通过两个阶段的消息传递来达成共识：提议阶段和接受阶段。Paxos 算法对于容忍节点故障具有强大的性能，但其理解和实现相对复杂。
* Raft 算法是一种相对较新且更易理解的分布式共识算法，其设计目标是提供与 Paxos 相同的一致性保证，但更容易理解和实现。Raft 将共识问题划分为领导选举、日志复制和安全性这三个相对独立的子问题，并通过限制选举过程中的竞争来简化了算法。

除了 Paxos 算法和 Raft 算法，针对不同的场景还有诸多不同的分布式共识算法。在实际使用中，选择特定算法取决于系统的需求、性能要求以及容忍的故障类型，需要根据具体场景做出合适的选择。
## 二、Paxos 算法核心
### 2.1 角色定义与提议

在 Paxos 算法中，有三个主要角色：提议者、接受者和学习者。这些角色共同协作以达成一致性，并且在具体的实现中，一个节点可能会同时充当多种角色。每个角色的定义和主要功能如下：

* 提议者：提议者的主要责任是向系统中的节点提交提议。提议者希望达成一致性，它会向一组接受者发送提议。提议者的提议可以包含一个值，而接受者将根据提议的情况来接受或拒绝该提议。
* 接受者：接受者的主要责任是接受或拒绝提议者的提议。接受者在接受到提议后，可以根据一定的规则来判断是否接受该提议。一个提议可能会被多个接受者接受，但最终只有一个提议能够在系统中达成一致。
* 学习者：学习者的任务是学习已经达成的一致性，即已经被接受的提议。学习者会从接受者那里获得已接受的提议，并将这些信息传播到整个系统中，以确保所有节点都知道已经达成的一致性。

> 一个提议可以同时被多个接受者接受，但最终系统只会选择其中的一个提议来达成一致性。尽管多个节点可能接受相同的提议，但系统需要最终选择一个提议来作为最终的共识值。这是因为在分布式环境中，存在并发的提议。多个提议者可以同时向系统提交提议，而多个接受者可能会接受其中的一个或多个提议。然后，系统必须经过一系列的协商和决策步骤，最终只选择一个提议作为系统的共识值。这确保了系统在分布式环境中的一致性。

在分布式系统中，提议是一种由提议者提交给接受者的操作，旨在达成分布式系统中的共识。提议通常包含一个值，提议者希望将这个值引入系统，并确保系统中的大多数节点都同意接受这个值，以达成一致性。提议通常包含两个元素，提议编号和提议值。每个提议都有一个唯一的编号，用于区分不同的提议，提议编号有助于解决并发提议的冲突，通常是一个单调递增的序列。提议值是提议者希望系统接受的实际值，这可以是任何系统状态的值，例如某个数据的更新。

提议的提交过程通常涉及多轮协商，以确保达到一致性。接受者可能会对提议进行投票，通过投票来决定是否接受提议，并通过与其他节点协商来达成共识。对于提议者而言，如果提议被半数以上的接受者接受，提议者就认为提议提交成功；对于接受者而言，只要接受者接受了某个提议，接受者就认为提议提交成功；对于学习者而言，接受者通知学习者哪个提议提交成功，学习者就认为提议提交成功。提议的提交过程会在下一小节中具体介绍，且会通过忽略学习者简化 Paxos 算法的运行过程。
### 2.2 提议的两阶段提交

Paxos 算法中的提议提交分为两个阶段，这两个阶段是为了确保在分布式系统中达成一致性。

在提议阶段中，提议者向所有的接受者发送一个提议，其中包含一个提议编号。每个接受者会在接受到提议后，判断当前是否已经接受了更高编号的提议。如果接受者没有接受过更高编号的提议，它会接受当前提议，并向提议者发送一个承诺，承诺不在接受小于该提议编号的提议。如果接受者已经接受了更高编号的提议，它会拒绝当前提议。

在接受阶段中，如果提议者收到了半数以上接受者的承诺，表示当前提议可以继续进行。提议者向所有的接受者发送一个接受请求，包含提议编号和提议值。接受者在接收到接受请求后，判断提议编号是否大于或等于它收到的所有提议阶段的提议编号，如果是，接受者接受当前提议，并通知其他节点，否则拒绝当前提议。

通过提议的两阶段提交，Paxos 保证了以下几点：每个提议都有一个唯一的提议编号，防止冲突；较大提议编号的提案优先被接受；提议只有在大多数节点接受的情况下才被最终接受，确保了系统的一致性。在第一阶段，提议者获得了对当前提议的承诺，确保没有更高编号的提议被接受。在第二阶段，只有在足够数量的承诺下，提议者才将提议者发送给接受者，最终确保只有一个值被系统接受，从而达成一致性。如果一个值被接受，那么所有正确的节点都会接受相同的值。这种两阶段提交的方式保证了分布式系统中的一致性，即使在节点故障或网络分区的情况下也能够可靠地工作。

> 提议值可以是任何想要在分布式系统中达成一致性的数据，例如状态更新、命令等。具体来说，提议者首先根据应用逻辑选择一个提议值。然而，由于分布式系统中存在网络延迟、消息丢失等问题，提议者可能需要通过与其他节点进行通信来达成一致性。在 Paxos 算法的运行过程中，提议者会根据其他节点的承诺和接受情况来确定最终的提议值。如果其他节点已经接受了一个提议，并且这个提议值比提议者最初选择的值更高（根据应用逻辑确定），那么提议者可能会选择接受这个更高的值，以保持系统的一致性。这种情况下，最终确定的提议值可能会受到其他节点的影响。
## 三、Paxos 算法两阶段实现
### 3.1 相关数据结构
#### 3.1.1 Proposal 结构体

```C++
struct Proposal {
    size_t serial_num;
    size_t value;
};
```

Proposal 结构体代表一个提议，serial_num 成员变量表示提议的提议编号，value 成员变量表示提议的提议值。
#### 3.1.2  Acceptor 类

```C++
class Acceptor {
public:
    Acceptor();
    bool propose(size_t serial_num, Proposal &last_accept_value);
    bool accept(Proposal &value);

private:
    size_t max_serial_num_;
    Proposal last_accept_value_;
};
```

Acceptor 类代表 Paxos 算法模拟中的单个接受者，它包含两个成员变量 max_serial_num_ 和 last_accept_value_。max_serial_num_ 记录在提议阶段接受的最大提议号，last_accept_value_ 记录该接受者接受的提议号最大的提议。Acceptor 类包含两个关键函数，propose 函数实现接受者在提议阶段的处理逻辑，accept 函数实现接受者在接受阶段的处理逻辑。
#### 3.1.3 Proposer 类

```C++
class Proposer {
public:
    Proposer(size_t proposer_count, size_t acceptor_count);
    void restart();
    bool proposed(bool ok);
    bool accepted(bool ok);
    bool is_propose_finished();
    bool is_accept_finished();

private:
    size_t proposer_count_;
    size_t acceptor_count_;
    bool is_propose_finished_;
    bool is_accept_finished_;
    size_t ok_count_;
    size_t refuse_count_;
};
```

Proposer 类代表 Paxos 算法模拟中的单个提议者。proposer_count_ 和 acceptor_count_ 记录 Paxos 算法模拟中的提议者和接受者的数量，is_propose_finished_ 和 is_accept_finished_ 记录提议者当前是否结束提议和接受阶段，ok_count_ 和 refuse_count_ 记录投票情况。Proposer 类包含三个关键函数，restart 函数用于在提议失败时重制相关配置，proposed 函数用于处理提议阶段的投票情况，accepted 函数用于处理接受阶段的投票情况。
### 3.2 Propose 阶段

```C++
bool Acceptor::propose(size_t serial_num, Proposal &last_accept_value) {
    if (serial_num == 0) {
        return false;
    }
    if (max_serial_num_ > serial_num) {
        return false;
    }
    max_serial_num_ = serial_num;
    last_accept_value = last_accept_value_;
    return true;
}
```

Acceptor 类的 propose 函数实现如上所示。接受者通过比较提议编号来决定是否投票支持支持提议，当提议编号小于接受者记录的在提议阶段接受的最大提议编号时，拒绝该提议，否则投票支持该提议。当接受者投票支持该提议时，接受者更新记录的在提议阶段接受的最大提议编号，并推荐该提议者最后一次接受的提议值。

```C++
bool Proposer::proposed(bool ok) {
    if (!ok) {
        refuse_count_++;
        if (refuse_count_ > acceptor_count_ / 2) {
            return false;
        }
        return true;
    }
    ok_count_++;
    if (ok_count_ > acceptor_count_ / 2) {
        ok_count_ = 0;
        refuse_count_ = 0;
        is_propose_finished_ = true;
    }
    return true;
}
```

Proposer 类的 proposed 函数实现如上。提议者在处理提议阶段的投票时，如果半数以上的接受者拒绝或着没有回应，则返回提议失败，需要开启新一轮提议。如果累计收到半数以上的支持投票，则结束提议阶段，开启接受阶段。
### 3.3 Accept 阶段

```C++
bool Acceptor::accept(Proposal &value) {
    if (value.serial_num == 0) {
        return false;
    }
    if (max_serial_num_ > value.serial_num) {
        return false;
    }
    last_accept_value_ = value;
    return true;
}
```

Acceptor 类的 accept 函数实现如上所示，如果接受者记录的在提议阶段支持的最大提议编号大于提议编号，说明接受者在提议阶段同意了其他提议编号更大的提议，则拒绝本次提议，否则同意本次提议，并更新接受者记录的最后接受的提议值。

```C++
bool Proposer::accepted(bool ok) {
    if (!ok) {
        refuse_count_++;
        if (refuse_count_ > acceptor_count_ / 2) {
            return false;
        }
        return true;
    }
    ok_count_++;
    if (ok_count_ > acceptor_count_ / 2) {
        ok_count_ = 0;
        refuse_count_ = 0;
        is_accept_finished_ = true;
    }
    return true;
}
```

Proposer 类的 accepted 函数实现如上。Proposer 类的 accepted 函数的处理与 Proposer 类的 proposed 函数类似，如果半数以上的批准者拒绝或没有回应，则需要开启新一轮提议。如果收到半数以上的支持投票，则结束接受阶段，达成一致。
### 3.4 Paxos 算法模拟

```C++
const int proposer_count = 5;
const int acceptor_count = 11;

std::mutex m[acceptor_count];
paxos::Acceptor a[acceptor_count];
std::mutex g_m;
size_t finish_count = 0;

void propose_loop(size_t index);

int main() {
    for (size_t i = 0; i < proposer_count; i++) {
        std::thread t(propose_loop, i);
        t.detach();
    }
    while (true) {
        if (finish_count == proposer_count) {
            break;
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }
    std::cout << "paxos simulation finished." << std::endl;
    return 0;
}
```

Paxos 算法模拟为每个提议者创建一个线程循环进行提议提交过程，并提供数个接受者全局变量。接受者全局变量通过多个互斥锁进行线程的同步，当所有提议者都通过相应的提议时结束模拟过程。提议者线程完整的线程循环实现如下：

```C++
void propose_loop(size_t index) {
    paxos::Proposer proposer(proposer_count, acceptor_count);
    paxos::Proposal proposal;
    proposal.serial_num = index + 1;
    proposal.value = index + 1;
    while (true) {
        size_t id[acceptor_count];
        size_t count = 0;
        paxos::Proposal last_proposal;
        for (size_t i = 0; i < acceptor_count; i++) {
            std::this_thread::sleep_for(std::chrono::milliseconds(std::rand() % 100));
            std::unique_lock<std::mutex> lock(m[i]);
            bool ok = a[i].propose(proposal.serial_num, last_proposal);
            lock.unlock();
            std::this_thread::sleep_for(std::chrono::milliseconds(std::rand() % 100));
            if (!proposer.proposed(ok)) {
                break;
            }
            id[count++] = i;
            if (proposer.is_propose_finished()) {
                break;
            }
        }
        if (!proposer.is_propose_finished()) {
            proposal.serial_num += proposer_count;
            proposer.restart();
            continue;
        }
        for (size_t i = 0; i < count; i++) {
            std::this_thread::sleep_for(std::chrono::milliseconds(std::rand() % 100));
            std::unique_lock<std::mutex> lock(m[id[i]]);
            bool ok = a[id[i]].accept(proposal);
            lock.unlock();
            std::this_thread::sleep_for(std::chrono::milliseconds(std::rand() % 100));
            if (!proposer.accepted(ok)) {
                break;
            }
            if (proposer.is_accept_finished()) {
                break;
            }
        }
        if (!proposer.is_accept_finished()) {
            proposal.serial_num += proposer_count;
            proposer.restart();
            continue;
        }
        std::lock_guard<std::mutex> lock(g_m);
        std::cout << "proposer " << index << " accepted." << std::endl;
        std::cout << "serial_num: " << proposal.serial_num << "." << std::endl;
        std::cout << "value: " << proposal.value << "." << std::endl;
        finish_count++;
        break;
    }
}
```
### 3.5 一些值得一提的点

（1）提议者线程循环会创建一个初始的提议编号和提议值。当提议者在提议提交的过程中失败时，会通过加上提议者总数来更新提议号。由于不同提议者线程循环的初始提议编号不同，这种处理方式简化了提议编号的全局唯一性。

（2）在模拟提议提交的过程中，提议者线程循环通过使线程随机睡眠一定时间来模拟与接受者信息传递过程中的时延和不稳定性。并且，提议者线程循环在处理接受阶段时，只会向在提议阶段支持提议的接受者发送提议接受请求。

（3）在提议阶段，支持提议的接受者会返回各个接受者目前已经接受的最大提议编号的提议值，提议者线程循环可以根据这些提议值选择一个可能更合理的提议值。在 Paxos 算法模拟中，希望每个提议者的初始提议值都可以提交，而不存在直接的大小关系，因此在实现中直接忽略了接受者返回的建议提议值。
## 四、Multi-Paxos 算法改进

Multi-Paxos 算法是对经典 Paxos 算法的扩展，用于提高在分布式系统中达成一致性的性能。Mutil-Paxos 算法相较于 Paxos 算法的主要改进在于以下几点：

* Multi-Paxos 算法允许一个节点在连续的提案中保持领导地位，而不需要在每个提案之前进行领导选举。这减少了领导选举的频率，降低了系统开销。
* Multi-Paxos 算法引入了提议流水线的概念，允许领导者并行地处理多个提议，这提高了系统的吞吐量，因为领导者可以同时处理多个提议，而不是等待一个提议完成后再处理下一个。
* Multi-Paxos 算法具有更高的可配置性，允许系统在不同的场景中调整参数以优化性能。这种灵活性使得 Multi-Paxos 算法能够适应不同的网络环境和负载状况。

Multi-Paxos 提高了分布式系统中一致性操作的性能和效率，使得它在需要高吞吐量和低延迟的场景中相对于传统的 Paxos 算法更具优势。