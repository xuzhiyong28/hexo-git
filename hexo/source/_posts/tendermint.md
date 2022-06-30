---
title: tendermint
date: 2022-05-18 10:04:46
tags:
---

### tendermint哨兵-Validator搭建模式

![](tendermint\1.png)

验证器将只与提供的哨兵通话，哨兵节点将通过秘密连接与验证器通信，网络的其余部分通过正常连接与验证器通信。哨兵节点也有相互通信的选项。`config.toml`初始化节点时，可能需要更改其中的五个参数：

- `mode:`(full | validator | seed) 节点模式（默认值：'full'）。如果要将节点作为验证器运行，请将其更改为“验证器”。
- `pex:`布尔值。这将打开或关闭节点的对等交换反应器。当 时`pex=false`，只有`persistent-peers`列表可用于连接。
- `persistent-peers:`一个逗号分隔的`nodeID@ip:port`值列表，定义了预期始终在线的对等点列表。这在第一次启动时是必要的，因为通过设置`pex=false`节点将无法加入网络。
- `unconditional-peer-ids:`逗号分隔的 nodeID 列表。无论入站和出站对等点的限制如何，这些节点都将连接。当哨兵节点有完整的地址簿时，这很有用。
- `private-peer-ids:`逗号分隔的 nodeID 列表。这些节点不会被传到网络上。这是一个重要的字段，因为您不希望您的验证者 IP 被传到网络。
- `addr-book-strict:`布尔值。默认情况下，将考虑具有可路由地址的节点进行连接。如果关闭此设置 (false)，则不可路由的 IP 地址（如专用网络中的地址）可以添加到通讯簿中。
- `double-sign-check-height`int64 高度。在加入共识之前要回顾多少块来检查节点的共识投票是否存在当非零时，如果使用相同的共识密钥签署最后一个块，节点将在重启时恐慌。因此，验证者应该停止状态机，等待一些块，然后重新启动状态机以避免恐慌。

#### Validator配置

| Config Option            | Setting                    |
| :----------------------- | :------------------------- |
| mode                     | validator                  |
| pex                      | false                      |
| persistent-peers         | list of sentry nodes       |
| private-peer-ids         | none                       |
| unconditional-peer-ids   | optionally sentry node IDs |
| addr-book-strict         | false                      |
| double-sign-check-height | 10                         |

要将节点作为验证器运行，mode=validator。验证器节点应该有pex=false，这样它就不会向整个网络散布谣言。`persistent-peers`将是你的哨兵节点。`private-peers`可以留空，因为验证器并不试图隐藏它在与谁通信。设置`unconditional-peer-ids`对于验证器来说是可选的，因为他们不会有完整的地址簿。

#### Sentry配置

| Config Option          | Setting                                       |
| :--------------------- | :-------------------------------------------- |
| mode                   | full                                          |
| pex                    | true                                          |
| persistent-peers       | validator node, optionally other sentry nodes |
| private-peer-ids       | validator node ID                             |
| unconditional-peer-ids | validator node ID, optionally sentry node IDs |
| addr-book-strict       | false                                         |

哨兵节点应该能够与整个网络通信，这是为什么`pex=true`。哨兵节点的`persistent peers`将是验证者，并且可选地是其他哨兵节点。哨兵节点应该确保他们不会泄露验证器的 ip，为此，您必须将验证器 nodeID 作为`private-peer`。无条件的对等点 ID 将是验证者 ID 和可选的其他哨兵节点。