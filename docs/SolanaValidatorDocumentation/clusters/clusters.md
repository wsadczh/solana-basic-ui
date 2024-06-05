# Solana 集群概述

Solana 集群是一组验证者（validators）节点一起工作以处理客户端交易并维护账本的完整性。许多集群可以共存。当两个集群共享一个共同的创世区块时，它们会尝试融合。否则，它们将忽略彼此的存在。发送到错误集群的交易会被悄然拒绝。在本节中，我们将讨论如何创建集群，节点如何加入集群，如何共享账本，如何确保账本的复制，以及如何应对有缺陷和恶意的节点。

## 创建集群

在启动任何验证者之前，首先需要创建一个创世配置（genesis config）。配置中引用了两个公钥，一个是铸造（mint）公钥，一个是引导验证者（bootstrap validator）公钥。持有引导验证者私钥的验证者负责将第一个条目添加到账本中。它用铸造账户初始化其内部状态。该账户将持有创世配置中定义的原生代币数量。然后，第二个验证者联系引导验证者以注册为验证者。其他验证者随后可以向集群中的任何已注册成员注册。

一个验证者会从领导者（leader）那里接收所有条目，并提交投票确认这些条目是有效的。投票后，验证者应存储这些条目。一旦验证者观察到足够多的副本存在，它就会删除自己的副本。

## 加入集群

验证者通过发送注册消息到集群的控制平面来加入集群。控制平面使用一种八卦协议（gossip protocol）实现，这意味着节点可以向任何现有节点注册，并期望其注册信息传播到集群中的所有节点。所有节点同步所需的时间与参与集群节点数量的平方成正比。从算法上讲，这是非常慢的，但作为交换，一个节点可以确保它最终拥有与每个其他节点相同的信息，而且这些信息不能被任何一个节点审查。

## 发送交易到集群

客户端将交易发送到任何验证者的交易处理单元（TPU）端口。如果节点的角色是验证者，它会将交易转发给指定的领导者。如果是领导者，则节点会打包传入的交易，为它们添加时间戳以创建条目，然后将它们推送到集群的数据平面上。一旦进入数据平面，交易就会由验证者节点进行验证，从而有效地将它们附加到账本中。

## 确认交易

一个 Solana 集群能够在几毫秒内确认数千个节点的交易，并计划扩展到数十万个节点。确认时间预计只会随着验证者数量的对数增加而增加，其中对数的底数非常高。例如，如果底数是一千，这意味着对于前一千个节点，确认时间将是三个网络跃点加上超多数验证者投票所需的时间。对于接下来的百万个节点，确认时间只增加一个网络跃点。

Solana 将 确认 定义为，从领导者为新条目打上时间戳到识别出超多数分类账投票之间的时间。

通过以下组合技术可以实现可扩展的  确认 ：

1. 使用 VDF 示例对事务进行时间戳，并对时间戳进行签名。

2. 将交易分成不同的组，发送给不同节点，并让每个节点与其对等节点共享其分组

3. 递归重复上一步骤，直到所有节点拥有所有分组。

Solana 在固定的间隔内轮换领导者，这些间隔称为槽（slots）。每个领导者只能在其分配的槽内生成条目。因此，领导者会为交易打上时间戳，以便验证者可以查找指定领导者的公钥。然后领导者签署时间戳，以便验证者可以验证签名，证明签名者是指定领导者公钥的拥有者。

接下来，交易被分组，以便节点可以将交易发送给多个方而无需制作多个副本。例如，如果领导者需要将 60 笔交易发送给 6 个节点，它会将 60 笔交易分成 10 笔交易的分组并发送给每个节点。这使得领导者可以在线路上传输 60 笔交易，而不是每个节点 60 笔交易。然后每个节点与其对等节点共享其分组。一旦节点收集了所有 6 个分组，它就会重建原始的 60 笔交易。

一个交易分组只能分割有限次数，因为分组过小会使头部信息成为网络带宽的主要消耗者。截至撰写本文时（2021 年 12 月），这种方法在大约 1250 个验证者中扩展良好。为了扩展到数十万个验证者，每个节点可以对另一组大小相等的节点应用与领导者节点相同的技术。我们称这种技术为 [*Turbine 块传播*](https://docs.solanalabs.com/consensus/turbine-block-propagation)（Turbine Block Propagation）。 