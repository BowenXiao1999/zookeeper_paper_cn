# ZooKeeper: wait-free 的大型网络协调系统

## Abstract

在这篇论文中，我们描述了 ZooKeeper，一个协调分布式应用的服务。ZooKeeper 是基础架构的一部分，目标是提供一个简单的、高性能的内核，供客户端构建更复杂的协调原语。它在多副本、中心化的服务中，组合了 group messaging, shared registers 和分布式锁服务。ZooKeeper 提供的接口有 wait-free 的 shared registers, 和一个与文件系统 cache 失效相似的事件驱动机制，用于提供一个简单但功能强大的协调服务。

ZooKeeper 接口使实现高性能服务成为可能。除了 wait-free 的属性之外，ZooKeeper 提供了一个每个客户端请求是 FIFO 执行的语义，和改变 ZooKeeper 状态的请求都是 Linearizable 的语义。这些设计决策可以使 ZooKeeper 的读可以读本地服务器，使实现高性能处理请求流水线成为可能。对于目标工作负载，我们显示了2:1到100:1 的读/写请求比例，ZooKeeper 每秒可以处理成千上万的事务。 这种性能使ZooKeeper可以被客户端应用程序广泛使用。

## 1. Introduction

大规模的分布式应用需要不同形式的协调。配置是最基础的形式之一。在它最简单的形式，系统进程中，配置是一个系统进程的的可操作的参数列表，更复杂的系统有动态的配置参数。Group membership 和领导选举也是分布式系统的共同有服务：通常，进程需要知道哪些别的进程状态是 alive 的和这些进程负责什么。Locks 建立了一个强大的同步原语，实现对关键资源的互斥访问。

一种协调的方式是，为不同的需求开发不同的服务。例如，Amazon Simple Queue Service \[3\] 专注于队列服务。其他服务被开发，用于特定的用处，比如领导选举和配置。实现了更强原语的服务可以被用于实现没有那么强大原语的服务。例如，Chubby 是一个有很强同步保证的锁服务。锁可以用来实现领导选举，组成员等服务。

当设计我们的协调服务的时候，我们抛弃了在服务端测实现特定的原语，作为替代，我们选择暴露一个能够使应用开发者实现他们自己同步原语的 API。这样的决定让我们实现一个使不需要改变服务而实现不同同步服务成为可能的*协调内核*。这种方法允许用户实现多种形式的协调，以适应应用程序的需求，而不是将开发人员限制在一组固定的原语上。

当设计 ZooKeeper 的 API 的时候，我们抛弃了阻塞原语，例如 Locks。对于一个协调服务，阻塞原语可以导致速度缓慢或故障的客户端等其他问题，从而对速度较快的客户端的性能产生负面影响。如果处理请求取决于响应和其他客户端的失败检测，则服务本身的实现将变得更加复杂。因此，我们的系统 ZooKeeper 实现了一个API，该API可以按文件系统的层次结构来组织简单的 wait-free 数据对象。 实际上，ZooKeeper API类似于任何其他文件系统，并且仅从 API 签名来看，ZooKeeper 似乎很像没有锁定方法，打开和关闭方法的 Chubby。 但是，实现 wait-free 数据对象使 ZooKeeper 与基于锁之类的阻塞原语的系统明显不同。

尽管无等待属性对于性能和容错性很重要，但不足以进行协调。我们还必须停工操作的顺序保证。特殊的，我们发现，对客户端所有操作提供 FIFO 语义与提供 *linearizable writes* 可以高效的实现服务，并且足以实现应用程序感兴趣的协调原语。实际上，对于使用API的任意数量的进程，都可以实现一致性，根据Herlihy给出的层次结构，ZooKeeper实现了全局对象。

ZooKeeper 服务用使用了复制来保证高可用和性能的服务器组成。它的高性能使包含大量进程的应用程序可以使用这种协调内核来管理协调的各个方面。我们能够使用一个简单的流水线架构，让我们能够处理成百上千的请求，同时仍然保持低延迟。这样的流水线很自然地可以保证对于单个客户端所有操作执行的顺序性。FIFO客户端顺序使得客户端可以异步提交操作请求。使用异步操作，客户端可以同时处理多个未完成操作。使用异步操作，客户端一次可以执行多个未完成的操作。这个功能是很棒的，例如，当新客户端成为领导者并且必须操纵元数据并相应地对其进行更新时。 如果不可能进行多个未完成的操作，则初始化时间可以是几秒左右，而不是亚秒级。

为了保证更新操作满足 linearizability，我们实现了一个基于 leader 的原子广播协议 Zab。然而一个典型的 ZooKeeper 应用，在支配地位的负载是读操作，所以需要保证读吞吐量的扩展性。在ZooKeeper中，服务器在本地处理读操作，并不需要使用Zab。

在客户端测缓存数据是提升读性能的重要技术。例如，对于一个进程，缓存现有 Leader 的 id 而不是每次需要 leader 都探测 leader 是很有效的。ZooKeeper 使用一种 watch 机制而不是直接操作缓存，来保证客户端缓存数据。有了 watch 机制，一个客户端可以 watch 一个给给定对象的更新请求，并接收到 update 的请求。（作为对比）Chubby 字节管理客户端的 cache，它会阻塞更新，以使更新部分的客户端的缓存全部失效。在这样的设计下，如果任何客户端响应慢或者出现错误，更新会被延迟。Chubby 使用 lease 机制防治一个客户端用用阻塞系统。但 leases 只能约束慢客户端或者错误客户端的影响，然后 ZooKeeper 的 watches 可以完全避免这个问题。

本论文讨论ZooKeeper的设计和实现，使用ZooKeeper，即使只有写入是可线性化的，我们也可以实现应用程序所需的所有协调原语。 为了验证我们的方法，我们展示了如何使用 ZooKeeper 实现一些协调原语。

作为总结，这篇文章中，我们主要的贡献是：

* **Coordination kernel**: 为了在分布式系统中使用，我们添了一个提供 relaxed 的一致性保证的 wait-free 的协作服务。特别是，我们描述了*协调内核*的设计和实现，我们已经在许多关键应用程序中使用了协调内核来实现各种协调技术。
* **Coordination recipes**: 我们展示了 ZooKeeper 如何可用于构建通常在分布式应用程序中使用的高级协调原语，甚至包含阻塞和强一致性原语。
* **Experience with Coordination**: 我们分享了一些使用 ZooKeeper 并评估其性能的方式。

## 2. The ZooKeeper Service

客户端通过 ZooKeeper client API library 向 ZooKeeper 提交请求。除了暴露 ZooKeeper 服务的 client API 接口, ZooKeeper Client library 也管理 client 和 ZooKeeper 服务器间的网络连接。

在这一届中，我们写提供一个 ZooKeeper 服务的 high-level view。然后我们讨论 client 与 ZooKeeper 操作的 API。

> **术语** 在这篇论文章，我们使用 *`client`* 来表示一个 ZooKeeper 服务的使用者，*`znode`* 表示一个内存中的 ZooKeeper 数据的节点。`znode` 会被组织成一颗被称为 `data tree` 的带层次的名称空间。我们还使用术语“更新和写入”来指代任何修改数据树状态的操作。 客户端在连接到 ZooKeeper 并获得 `session` 句柄以发出请求时建立 `session`。

### 2.1 Service Overview

ZooKeeper 给它的客户端提供了“data nodes\(`znodes`\)的集合”的抽象，用层次的名称空间组织他们。client 通过 API 操纵在这个层次中的数据对象。层次的名称空间在文件系统里被广泛使用。这是一种组织层次空间的可靠方法，因为用户习惯这种抽象，同时它使更好的应用元数据组织成为可能。为了引用一个给定的 `znode` ，我们使用标准的 UNIX 文件系统路径。例如，我们使用 `/A/B/C` 来表示到 `znode` C 的路径，B 是 C 的父节点，同时 A 是 B 的父节点。所有的 `znode` 都存储数据，同时，除了 `ephemeral znodes` 外的所有的 `znode` ，都能拥有子节点。

client 可以创建两种 `znode`:

* `Regular`: client 显式操纵和删除 `regular znodes`
* `Ephemeral`: client 会创建这种节点，它可以被显式唱出，也可以在创建这个节点的 `session` 断开的时候自动删除（故意或由于故障）。

此外，当我们创建一个新的 `znode` 的时候，一个 client 可以设置 `sequential` flag. 通过 `sequential` flag 创建的节点会有在名称添加一个单调递增的计数器。如果 `n` 是新的 `znode`, `p` 是他的父节点，那么 `n` 的名称中添加的值不会小与 `p` 的子节点中创建过的任何一个添加的值。

ZooKeeper 通过现实 watch 来允许 client 不需要轮询的定时接收到值的变化的通知。当一个客户端带有 `watch` flag 并发起读请求的时候，该操作将正常完成，除了服务器承诺在返回的信息已更改时会通知客户端。 监视是与会话相关的一次性触发器； 一旦触发或会话关闭，它们将不被注册。 监视表明发生了更改，但未提供更改。 例如，如果客户端在两次更改`/foo`之前发出了请求 `getData('/foo'，true)`，则客户端将只获得一个 watch事件，告知客户端`/foo`的数据已更改。 `session` 事件（例如连接丢失事件）也将发送到 watch 回调，以便客户端知道 watch 事件可能会延迟.

#### 数据模型

ZooKeeper 的数据模型是一个只有把全部数据整个读/写的文件系统的简化 api, 或者有 key 层次的 key/value 表。层次名称空间对于为不同应用程序的名称空间分配子树以及设置对这些子树的访问权限很有用。 我们还将在客户端利用目录的概念来构建更高级别的原语，如我们在2.4节中将看到的。

![fig01_hierarchical_name_space](images/fig01_hierarchical_name_space.png)

与文件系统中的文件不同，znode不适用于常规数据存储。 相反，`znodes` 存储 client 引用的数据，通常是用于协调的元数据。为了图解，在 Figure 1 中我们有两个子树，一个用于应用程序1（`/app1`），另一个用于应用程序2（`/app2`）。应用程序1的子树实现了一个简单的组成员身份协议：每个客户端进程`pi`在`/app1`下创建一个`znode`  `pi`，只要该进程正在运行，该节点便会持续存在。

尽管 `znode` 并非设计用于常规数据存储，但是 ZooKeeper 允许客户端存储一些可用于分布式计算中的元数据或配置的信息。例如，在基于 leader 的应用程序中，这对于刚刚开始了解哪个其他服务器当前是领导者的应用程序服务器很有用。为了实现此目标，我们可以让当前的领导者在znode空间中的已知位置写入此信息。 `znode` 还将 元数据与 timestamp 和 version counter 关联，这使客户端可以跟踪对 `znode` 的更改并根据 `znode` 的版本执行有条件的更新。

#### Sessions

client 连接到 ZooKeeper 并初始化 `session`。`session `具有关联的 timeout。如果ZooKeeper 在 timeout 内没有收到来自 `session` 的 client 的任何消息，则认为该 client 有故障。 当 client 显式关闭会话句柄或 ZooKeeper 检测到 client 故障时，会话结束。 在会话中，客户端观察到一系列状态变化，这些状态变化反映了其操作的执行。 会话使客户端可以在ZooKeeper集成中从一台服务器透明地移动到另一台服务器，因此可以在ZooKeeper服务器之间持久存在。在 `session` 中，client 观察到一系列状态变化，这些状态变化反映了 ZooKeeper 操作的执行。 `session` 使客户端可以在 ZooKeeper 集群中从一台服务器透明地移动到另一台服务器，因此`session`可以在 ZooKeeper 服务器之间持久存在。

### 2.2 Client API

我们在下方展示了一个 ZooKeeper API 的子集，并讨论了每个请求的语义：

* `create(path, data, flags)`: 根据路径名称 `path`，它存储的`data[]`，创建一个 `znode`, 并返回这个新的 `znode` 的名称。`flags` 允许客户端选择选定的 `znode` 类型：`regular`, `ephemeral` 及设置 `sequential`  flag。
* `delete(path, version)`: 如果 `znode` 符合给定的 `version` 版本，则删除`path` 下的 `znode`。
* `exists(path, watch)`: 如果 `path` 下的 `znode` 存在，返回 true, 否则返回 false.`watch` 标志可以使 client 在 `znode` 上设置 watch。
* `getData(path, watch)`: 返回 `znode` 的数据和元数据（元数据例如版本信息）。`watch` 和 `exists()` 里面的作用一样，不同之处在于，如果`znode`不存在，则 ZooKeeper 不会设置手表。
* `setData(path, data, version)`: 如果 `version` 是 `znode` 现有的版本，把 `data[]` 写进 `znode`.
* `getChildren(path, watch)`: 返回`path` 对应的 `znode` 的子节点集合。
* `sync(path)`: 等待操作开始时所有没有同步的更新传播到 client 连接到的服务器。 该`path` 当前被忽略。

所有的的方法在 API 中都有一个同步版本和一个异步版本。 当应用程序需要执行单个 ZooKeeper 操作且没有要并发执行的任务时，它会使用同步API，因此它会进行必要的 ZooKeeper 调用并进行阻塞。但是，异步API使应用程序可以并行执行多个 ZooKeeper 操作和其他任务。 ZooKeeper client 保证按顺序调用每个操作的相应回调。

需要注意的是，ZooKeeper 不使用句柄来操纵 `znode`. 作为替代，每个请求都带有需要操作的 `znode` 的完整路径。这样不知简化了 API \(没有 `open()` 和 `close()` 方法\), 也消除了服务器需要维护的额外状态。

每种更新方法均需要一个预期的 version，从而可以实施条件更新。 如果 `znode` 的实际版本号与预期版本号不匹配，则更新将失败，并出现 ` unexpected version error`。 如果给定的预期版本号为 `-1`，则不执行版本检查。

### 2.3 ZooKeeper guarantees

ZooKeeper 有两个基本的顺序保证：

* **Linearizable Writes**: 所有的更新 ZooKeeper 状态的请求都是 serializable 的，并且遵循优先级。
* **FIFO client order**: 一个给定客户端发送的所有的请求都是按照客户端发送的顺序有序执行的。

注意我们对 `linearizability` 的定义和 Herihy 的原始提议不同，我们叫它 *A-linearizability* \(asynchronize linearizability\). 在 Herilihy 的原始的 linearizable 顶柜中，一个客户端只能在同一时间内有一个未完成的请求（一个客户端是一个线程）。在我们的系统中，我们允许一个客户端有复数个未完成的操作，并且我们可以选择不保证未完成操作的执行顺序，或者使用保证 FIFO 顺序。我们选择后者作为我们的属性，可以观察到，对于保持 linearizable 对象也会保持 A-linearizable ，因为满足 A-linearizable 的系统也会满足 linearizable。因为只有更新请求是 A-linearizable 的，所以 ZooKeeper 可以在每个副本本地处理读请求。这样，当服务器添加到系统时，服务可以线型扩展。

为了了解两个保证是怎么讲i傲虎的，考虑下列的场景。一个系统由一组进程组成，它选择出一个 leader，并由 leader 操作工作进程。但一个新的 Leader 接管整个系统时，它必须更新大量的配置参数，并在更新结束的时候通知其它进程。这样我们有两个重要的需求：

1. 当新的 Leader 开始更改系统时，我们不希望其它进程开始使用被更改的配置。
2. 当新的 Leader 在配置完全更新完成之前就宕机时，我们不希望其它进程使用半更新的配置。

观察到分布式锁，例如 Chubby 提供的锁，将会帮助我们完成需求1，但是对需求2并不有效。有了 ZooKeeper, 新的 Leader 可以指定一个路径，并把它当成一个 `ready znode`，其他的进程只会在这个 `znode` 存在的时候使用这套配置。新的 leader 靠 \(1\) 删除 `ready` \(2\) 更新配置的 `znode` \(3\) 重新创建 `ready` 来完成上述需求。上述的所有变更可以被流水线处理，并发起一个异步请求来快速更新配置状态。尽管更改操作的等待时间约为2毫秒，但是如果一个请求接一个发出，则必须更新5000个不同znode的新领导者将花费10秒。 通过异步发出请求，请求将花费不到一秒钟的时间。由于顺序保证，如果进程看到 ready `znode`，则它还必须看到新的 Leader 所做的所有配置更改。 如果新的 Leader 在创建 ready `znode` 之前宕机，则其他进程知道该配置尚未完成，因此不使用它。

上述的模式仍然有一个问题：如果一个进程在新的 Leader 开始变更并读到配置看到 ready 存在会发生什么？这个问题通过 notification 的顺序保证解决：如果一个客户端正在 watch 一个变更，这个客户端会在它看到配置变更之前收到一个通知。因此，如果读取 ready `znode` 的进程请求要在 `znode` 变更的时候被通知，它的 client 会在收到配置变化之前，收到  ready `znode` 变化的通知。

当 client 除了 ZooKeeper之外还拥有自己的通信 channel 时，可能会出现另一个问题。 例如，考虑两个客户端 A、B在 ZooKeeper 中有共享的配置，并通过一个共享的通信 channel 通信。如果 A 更改了 ZooKeeper 中的共享配置，并通过 channel 告知 B，B 会期望在重新读取的时候看到配置的变化。 如果B的ZooKeeper副本稍微落后于A，则可能看不到新配置（因为读是在本地进行的）。使用上一段中的保证 B 可以通过在重新读取配置之前发出写入操作来确保它能看到最新信息。为了更有效地处理这种情况，ZooKeeper提供了 `sync` 请求：在进行读取后，构成了慢的读取读取。`sync`操作不需要一次写操作就可以在读之前使得服务器将所有 pending 的写请求完成。这一原语与ISIS的原语`flush`类似。

ZooKeeper 也有下述两个 liveness 和持久性保证：如果大部分 ZooKeeper 服务器是活跃的并且可以通信，ZooKeeper 服务是可用的， 如果 ZooKeeper 服务成功响应更改请求，任何数量的 ZooKeeper 服务器故障中，该变更都会持续存在，只要最终大部分服务器可以恢复。

### 2.4 Examples of primitives

在本节中，我们将展示如何使用 ZooKeeper API来实现更强大的原语。 ZooKeeper 服务对这些更强大的原语一无所知，因为它们是完全使用 ZooKeeper client API在客户端上实现的。 一些常见的原语（例如group membership 和配置管理）也是 wait-free 的。 对于其他地方，例如 rendezvous，client 需要等待事件。 即使ZooKeeper 无需等待，我们也可以使用 ZooKeeper 实现高效的阻塞原语。ZooKeeper的顺序保证允许对系统状态进行有效的推理，而 watch 则可以进行有效的等待。

#### 配置管理

ZooKeeper 可以被用来实现分布式应用中的动态配置。在它最简单的形式中，配置被存储在 `znode` `Zc` 中， 进程以 `Zc` 的完整路径名启动。 启动进程通过将 watch 标志设置为 true 来读取 `Zc` 来获取其配置。 如果`Zc`中的配置出现任何更新，则会通知进程并读取新配置，进程再次将watch标志设置为true。

注意在这个模式中，如同大部分其它使用 watch 的模式，watch 被用来保证进程有最近的信息。例如，如果一个 watch `Zc` 的进程，在读取之前发出了3个对 `Zc` 的更改，那么这个进程不会收到三个通知。这不会影响进程的行为，因为这三个事件只会通知进程它已经知道的内容：它对`Zc` 拥有的信息是陈旧的。

\(这里我有点不明白，是指更新的时候会完成一次 sync 么)

#### Rendezvous

有时在分布式系统中，我们不会预先知道最终的系统配置会是什么样子的。例如，一个客户端可能需要启动一个 master 进程和不少 worker 进程，但是启动进程是由调度器完成的，所以客户端不能提前知道知道需要连接的 Address 和 Port 等连接 master 需要的相关的信息。我们通过 client 创建 `rendezvous znode` （即 `Zr`）来解决这个问题。客户端把 `Zr` 的整个路径作为启动参数传给 master 和 worker 进程。当 master 启动时，它会把自己的地址和端口信息填充进 `Zr`.当 workers 启动的时候，它们会以 watch flag 读取 `Zr`。如果 `Zr` 还没有被填充， worker 就会等待 `Zr` 被更新。如果 `Zr` 是一个 `ephemeral` 节点，master 和 worker 进程可以通过 watch `Zr` 等待是否被删除，决定在 client 终止后对自己进行清理。

#### Group Membership

我们利用 `ephemral` 节点来实现 group membership. 特别的，我们利用 `ephemeral` 节点允许我们看到创建 `znode` 的 `session` 状态的特性。我们从指定一个 `znode` `Zg`，来代表这个 group 开始。当一个group的成员启动后，它在 `Zg` 下创建一个临时节点。如果每个进程都有一个唯一的名称或者标识符，那么这个名称就会被用作创建的子 `znode` 的名称; 否则，他会用 `SEQUENTIAL` flag 来创建一个有独立名称的 `znode`。进程可以将进程信息，入该进程使用的地址和端口，放入子`znode`的数据中。

在`Zg`下创建子`znode`后，该过程将正常启动。 它不需要做任何其他事情。 如果该过程失败或结束，则代表`Zg`的`znode` 会自动删除。进程可以通过简单列出`Zg`的子代来获取组信息。 如果某个进程希望监视组成员身份的更改，则该进程可以将监视标志设置为 true，并在收到更改通知时刷新组信息（始终将监视标志设置为true）。

#### Simple Locks

尽管ZooKeeper不是锁服务，但可以用来实现锁。使用 ZooKeeper 的应用程序通常使用根据其需求量身定制的同步原语，例如上面所示的那些。在这里，我们展示了如何使用 ZooKeeper 实现锁，以表明它可以实现各种各样的常规同步原语。

最简单的锁实现使用“lock files”。该锁由 `znode` 表示。为了获取锁，客户端尝试使用`EPHEMERAL`标志创建指定的`znode`。如果创建成功，则客户端将持有该锁。否则，客户端可以设置 watch 标志读取`znode`，以便在当前领导者去世时得到通知。客户端宕机或显式删除`znode`时会释放该锁。其他等待锁定的客户端一旦观察到`znode`被删除，就会再次尝试获取锁定\(即创建 `znode` )。

尽管此简单的锁定协议有效，但确实存在一些问题。首先，它具有惊群效应。如果有许多等待获取锁的客户端，则即使只有一个客户端可以获取锁，他们也会争夺该锁。其次，它仅实现互斥锁。以下两个原语显示了如何同时解决这两个问题。

#### Simple Locks without Herd Effect

我们定义一个锁`znode` *l* 来实现这种锁。 直观地说，我们排队所有请求锁定的客户端，每个客户端都按照请求到达的顺序获得锁定。 因此，希望获得该锁的客户端执行以下操作：

```
Lock
1: n - create(1 + "/lock-", EPHEMERAL | SEQUENTIAL)
2: C = getChildren(1, false)
3: if n is lowest znode in C, exit
4: p = znode in C ordered just before n
5: if exists(p, true) wait for watch event
6: goto 2

Unlock
1: delete(n)
```

在Lock的第1行中使用`SEQUENTIAL`标志，命令 client 尝试获取锁，并相对其它的尝试获得一个序列号。如果客户端的`znode`在第3行的序列号最小，则客户端将持有该锁。否则，客户端将等待删除 持有 Lock 或将在此客户端的`znode` 之前收到锁定的有更小序列号的 `znode` 。通过仅查看客户端`znode`之前的`znode`，我们仅在释放锁或放弃锁请求时才唤醒一个进程，从而避免了惊群效应。客户端监视的znode消失后，客户端必须检查它现在是否持有该锁。（先前的 Lock 请求可能已被放弃，并且具有较低序号的`znode` 仍在等待或保持锁定。）

释放锁定就简单地直接删除代表 Lock 请求的`znode`  *n*。通过在创建 `znode` 时使用`EPHEMERAL`标志，崩溃的进程将自动清除所有锁定请求或释放它们可能拥有的任何锁定。

总之，此锁定方案具有以下优点：

1. 删除一个`znode`只会导致一个客户端唤醒，因为每个`znode`都被另一个客户端 watch，因此我们没有惊群效应；
2. 没有轮询或超时；
3. 由于我们实现了锁定的方式，因此通过浏览ZooKeeper数据可以看到 Lock 争用，中断锁定和调试锁定问题的进程的数量。

#### Read/Write Locks

为了实现读/写锁，我们略微更改了锁过程，并分别设置了读锁和写锁。 解锁过程与全局锁定情况相同。

```
Write Lock:
1: n = create(1 + "write-", EPHEMERAL|SEQUENTIAL)
2: c = getChildren(1, false)
3: if n is lowest znode in C exit
4: p = znode in C ordered just before n
5: if exists(p, true) wait for event
6: goto 2
```



```
Read Lock
1: n = create(1 + "read-", EPHEMERAL|SEQUENTIAL)
2: c = getChildren(1, false)
3: if no write znodes lower than n in C, exit
4: p = write znode in C ordered just before n
5: if exists(p, true) wait for event
6: goto 3
```

该锁定过程与之前的锁定略有不同。 写锁仅在命名上有所不同。 由于读锁可能是 shared 的，因此第3行和第4行略有不同，因为只有较早的写锁`znode`会阻止客户端获得读锁。 当有多个客户端等待读锁时，当删除具有较低序号的`write-` znode时，我们可能会收到“惊群效应”。 实际上，这是一种期望的行为，所有那些需要读的客户端都应被通知，因为它们现在可能已具有锁定。

#### Double Barrier

double barriers 使 client 能够同步计算的开始和结束。当有 barrier 限制定义的足够多的进程加入 barrier 时，进程将会开始它们的计算，并在计算结束后离开 barrier。我们在 ZooKeeper 中用`znode` 表示一个 barrier，称为*b*。 每个进程p都会在进入时通过将`znode`创建为 *b* 的子节点来向 *b* 注册，并在准备离开时取消注册，即删除该子节点。 当b的子`znode`数量超过障碍阈值时，进程可以进入 barrier。 当所有进程都删除了其子进程时，进程可以离开 barrier。 我们使用 watch 来有效地等待进入和退出 barrier 条件得到满足。 要进入 barrier，流程会监视是否存在 *b* 的 ready 子 `znode` ，该子 `znode` 将由导致子节点数超过障碍阈值的进程创建。 要离开 barrier，进程会 watch 特定的子节点的消失，并且仅在这个`znode`被删除之后检查退出条件。

## 3 ZooKeeper Applications



## 4 ZooKeeper Implementation



### 4.1 Request Processor



