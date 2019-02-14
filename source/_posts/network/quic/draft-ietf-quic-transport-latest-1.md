---
title: QUIC协议翻译 - draft-ietf-quic-transport-latest Part 1.
date: 2019-02-13 11:09:28
tags:
  - QUIC
  - translate
  - RFC
  - Protocol
---
本文是对QUIC协议IETF提案：[draft-ietf-quic-transport-latest](https://quicwg.org/base-drafts/draft-ietf-quic-transport.html)翻译的第一部分。

[draft-ietf-quic-transport-latest](https://quicwg.org/base-drafts/draft-ietf-quic-transport.html)是在[draft-ietf-quic-invariants-latest](https://quicwg.org/base-drafts/draft-ietf-quic-invariants.html)基础上定义的第一个版本的QUIC协议，这里描述了QUIC协议的核心，包括一些概念定义与具体的协议行为。

第一部分翻译内容包括第1，2，3节


## Abstract

本文档定义了QUIC传输协议的核心。随附文档描述了QUIC的丢失检测和拥塞控制以及使用TLS进行密钥协商。

<!-- more -->

### 1. 简介

QUIC是一种多路复用且安全的通用传输协议，它具有如下特性：

- 流复用
- 流和连接级控制
- 低延迟连接建立
- 连接迁移和弹性的NAT重绑定
- 经过加密与认证的包头和负载

QUIC使用UDP作为基础，以避免需要更改旧版客户端操作系统和中间件。 QUIC验证其所有包头并加密它交换的大多数数据，包括其信令，以避免引起对中间件的依赖。

#### 1.1 文档结构

本文档描述了核心QUIC协议，其结构如下。

- 流是QUIC提供的基本服务抽象。
  - **第2节** 描述了与流相关的核心概念，
  - **第3节** 提供了流的参考模型，
  - **第4节** 概述了流量控制的操作。
- 连接是QUIC端点进行通信的主体。
  - **第5节** 描述了流相关的核心概念，
  - **第6节** 描述了版本协商，
  - **第7节** 详细描述了建立连接的过程，
  - **第8节** 明确关键的拒绝服务缓解机制，
  - **第9节** 描述了端点如何迁移连接到新的网络路径上，
  - **第10节** 列出了结束已经打开的流的选项，
  - **第11节** 提供了对错误处理的通用指导
- 数据包和数据帧是QUIC通讯的基础单元。
  - **第12节** 描述了数据包和数据帧相关的概念，
  - **第13节** 定义了传输，重传，和确认的模型
  - **第14节** 明确了管理数据包大小的规则
- 最后，QUIC协议元素的编码细节描述如下：
  - **第15节** （版本）
  - **第16节** （整数编码）
  - **第17节** （数据包头）
  - **第18节** （传输参数）
  - **第19节** （数据帧）
  - **第18节** （错误）

随附文档描述了QUIC的丢失检测和拥塞控制[QUIC-RECOVERY]，以及使用TLS进行密钥协商[QUIC-TLS]。

本文档定义了QUIC版本1，它符合[QUIC-INVARIANTS]中的协议不变量。

#### 1.2 公约和定义

> 略

#### 1.3 符号约定

本文档中的数据包和框架图使用[RFC2360]第3.1节中描述的格式，并附带以下附加约定：

```
[x]: 表示x是可选的
x(A): 表示x有A bit长
x(A/B/C): 表示x有A，B或C bit长
x(i): 表示x使用第16节描述的可变长编码
x(*): 表示x是可变长度的
```

### 2. 流

QUIC中的流为应用程序提供轻量级，有序的字节流抽象。 从另一个角度来说流是QUIC作为弹性的消息抽象。

可以通过发送数据来创建流。与流管理相关的过程（结束，取消和流控制）都旨在实现最小的开销。例如，单个STREAM帧可以打开流，携带数据或关闭流。流也可以是长期存在的，并且可以持续整个连接的持续时间。

流可以由任一端点创建，可以同时发送与其他流不同的数据，并且可以被取消。 QUIC没有提供任何确保不同流之间字节排序的方法。

QUIC允许任意数量的流同时运行，并且可以在任何流上发送任意数量的数据，受流控制约束（参见第4节）和流限制。

#### 2.1 流类型与识别码

流可以是单向的或双向的。单向流在一个方向上传输数据：从流的发起者到对等方。双向流允许数据在两个方向上发送。

通过数值在连接中标识流，称为流ID。流ID对于流是唯一的。 QUIC端点绝不能重用连接中的流ID。流ID被编码为可变长度整数（参见第16节）。

流ID的最低有效位（0x1）标识流的发起者。客户端发起的流具有偶数编号的流ID（位设置为0），服务器发起的流具有奇数编号的流ID（位设置为1）。

流ID的第二个最低有效位（0x2）区分双向流（位设置为0）和单向流（位设置为1）。

因此，来自流ID的最低有效两位将流识别为四种类型之一，如表1中所总结的。

| 掩码 | 发起者     | 流类型 |
| ---- | ---------- | ------ |
| 0x1  | 客户端发起 | 双向流 |
| 0x2  | 服务端发起 | 双向流 |
| 0x3  | 客户端发起 | 单向流 |
| 0x4  | 服务端发起 | 单向流 |

在每种类型中，使用数字增加的流ID创建流。不按顺序使用的流ID将会导致该类型的所有具有较低编号的流ID的流也被打开。

客户端打开的第一个双向流的流ID为0。

#### 2.2 接收和发送数据

STREAM帧封装了应用程序发送的数据。端点使用STREAM帧中的Stream ID和Offset字段按顺序放置数据。

端点必须能够将流数据作为有序字节流传递给应用程序。提供有序的字节流要求端点缓冲任何无序接收的数据，直到宣称流量控制限制。

QUIC没有特别规定无序传输流数据。但是，实现可以选择提供将数据无序传递到接收应用程序的能力。

端点可以多次接收相同流偏移的流的数据。已经收到的数据可以被丢弃。如果多次发送，给定偏移量的数据不得改变;端点可以将流中相同偏移量的不同数据的接收视为PROTOCOL_VIOLATION类型的连接错误。

Streams是一种有序的字节流抽象，QUIC中不存在其他可见的结构。当数据传输，数据包丢失后重传数据或者数据在接收器传送到应用程序时，预计不会保留STREAM帧边界。【TODO】

端点不得在任何流上发送数据，而不确保它在同级设置的流量控制限制范围内。流量控制在第4节中详细描述。【TODO】

#### 2.3 流优先级

如果分配给流的资源被正确地优先化，则流复用可以对应用性能产生显着影响。

QUIC不提供交换优先级信息的框架。相反，它依赖于从使用QUIC的应用程序接收优先级信息。

QUIC实现应该提供应用程序可以指示流的相对优先级的方式。在决定将资源专用于哪些流时，实现应该使用应用程序提供的信息。

### 3. 流状态

本节按发送或接收组件描述流。描述了两个状态机：一个是使用流传输数据时端点的状态机（第3.1节），一个是使用流接收数据时端点的状态机（第3.2节）。

单向流直接使用适用的状态机。双向流使用两个状态机。在大多数情况下，无论流是单向还是双向，这些状态机的使用都是相同的。打开流的条件对于双向流来说稍微复杂一些，因为发送侧或接收侧的打开导致流在两个方向上打开。

端点必须按流ID的递增顺序打开相同类型的流。

> 备注：这些状态机的信息很大。本文档使用流状态来描述何时以及如何发送不同类型的帧以及在接收到不同类型的帧时预期的反应的规则。虽然这些状态机旨在用于实现QUIC，但这些状态并非旨在约束实现。实现可以定义不同的状态机，只要其行为与实现这些状态的实现一致即可。

#### 3.1 发送流状态

图1显示了将数据发送到对等方的流的一部分的状态。

```
       o
       | Create Stream (Sending)
       | Peer Creates Bidirectional Stream
       v
   +-------+
   | Ready | Send RESET_STREAM
   |       |-----------------------.
   +-------+                       |
       |                           |
       | Send STREAM /             |
       |      STREAM_DATA_BLOCKED  |
       |                           |
       | Peer Creates              |
       |      Bidirectional Stream |
       v                           |
   +-------+                       |
   | Send  | Send RESET_STREAM     |
   |       |---------------------->|
   +-------+                       |
       |                           |
       | Send STREAM + FIN         |
       v                           v
   +-------+                   +-------+
   | Data  | Send RESET_STREAM | Reset |
   | Sent  |------------------>| Sent  |
   +-------+                   +-------+
       |                           |
       | Recv All ACKs             | Recv ACK
       v                           v
   +-------+                   +-------+
   | Data  |                   | Reset |
   | Recvd |                   | Recvd |
   +-------+                   +-------+
```

端点启动的流的发送部分（客户端类型0和2，服务器类型1和3）由应用程序打开。 `Ready`状态表示新创建的流，该流能够接受来自应用程序的数据。可以在此状态下缓冲流数据以准备发送。

发送第一个STREAM或STREAM_DATA_BLOCKED帧会导致流的发送部分进入`Send`状态。实现可以选择直到它发送第一帧并进入该状态时再将流ID分配给流，这可以允许更好的流优先级。

由对等方发起的双向流的发送部分（服务器类型0，客户端类型1）进入`Ready`状态时，如果接收部分进入`Recv`状态则立即转换到`Send`状态（第3.2节）。

在`Send`状态中，端点通过STEAM帧发送（在必要时重新发送 ）流数据。端点遵守其对等方设置的流量控制限制，并继续接受和处理MAX_STREAM_DATA帧。如果它被流量控制阻塞通过流或者连接发送数据，则`Send`状态的端点会生成STREAM_DATA_BLOCKED帧。

在应用程序指示已发送所有流数据并且发送包含FIN位的STREAM帧之后，流的发送部分进入`Data Sent`状态。从该状态开始，端点仅在必要时重传流数据。端点不需要检查流控制限制，也不需要为此状态的流发送STREAM_DATA_BLOCKED帧。可以接收MAX_STREAM_DATA帧，直到对等体接收到最终流偏移。端点可以安全地忽略它从对等端接收到的处于此状态的流的任何MAX_STREAM_DATA帧。

一旦成功确认了所有流数据，流的发送部分就进入`Data Recvd`状态，这是一种结束状态。

从任何`Ready`，`Send`或`Data Sent`状态，应用程序可以发信号通知它希望放弃流数据的传输。或者，端点可能从其对等端接收STOP_SENDING帧。在任何一种情况下，端点都会发送RESET_STREAM帧，这会导致流进入`Reset Sent`状态。

端点可以发送RESET_STREAM作为提及流的第一帧，这会导致该流的发送部分打开，然后立即转换到`Reset Sent`状态。

一旦确认了包含RESET_STREAM的分组，则流的发送部分进入`Reset Recvd`状态，这是终结状态。

#### 3.2 接收流状态

图2显示了从对等方接收数据的流部分的状态。接收流的一部分的状态仅镜像对等体流的发送部分的一些状态。流的接收部分不跟踪发送部分上无法观察到的状态，例如`Ready`状态。相反，流的接收部分跟踪向应用程序传送数据，其中一些数据不能被发送者观察到。

```
       o
       | Recv STREAM / STREAM_DATA_BLOCKED / RESET_STREAM
       | Create Bidirectional Stream (Sending)
       | Recv MAX_STREAM_DATA / STOP_SENDING (Bidirectional)
       | Create Higher-Numbered Stream
       v
   +-------+
   | Recv  | Recv RESET_STREAM
   |       |-----------------------.
   +-------+                       |
       |                           |
       | Recv STREAM + FIN         |
       v                           |
   +-------+                       |
   | Size  | Recv RESET_STREAM     |
   | Known |---------------------->|
   +-------+                       |
       |                           |
       | Recv All Data             |
       v                           v
   +-------+ Recv RESET_STREAM +-------+
   | Data  |--- (optional) --->| Reset |
   | Recvd |  Recv All Data    | Recvd |
   +-------+<-- (optional) ----+-------+
       |                           |
       | App Read All Data         | App Read RST
       v                           v
   +-------+                   +-------+
   | Data  |                   | Reset |
   | Read  |                   | Read  |
   +-------+                   +-------+
```

当为该流接收到第一个STREAM，STREAM_DATA_BLOCKED或RESET_STREAM时，将创建由对等方发起的流的接收部分（客户端的类型1和3，或服务器的0和2）。对于由对等方发起的双向流，接收流的发送部分的MAX_STREAM_DATA或STOP_SENDING帧也会创建接收部分。流的接收部分的初始状态是`Recv`。

当端点发起的双向流的发送部分（客户端类型0，服务器类型1）进入`Ready`状态时，流的接收部分进入`Recv`状态。

当从该流的对等体收到MAX_STREAM_DATA或STOP_SENDING帧时，端点将打开双向流。为未打开的流接收MAX_STREAM_DATA帧表示远程对等体已打开流并提供流控制分数。为未打开的流接收STOP_SENDING帧表示远程对等体不再希望接收此流上的数据。如果数据包丢失或重新排序，则任何一个帧都可能在STREAM或STREAM_DATA_BLOCKED帧之前到达。

在创建流之前，必须创建具有较低编号的流ID的所有相同类型的流。这可确保流的创建顺序在两个端点上保持一致。

在“Recv”状态中，端点接收STREAM和STREAM_DATA_BLOCKED帧。传入的数据被缓冲，可以重新组合成正确的顺序以便传送到应用程序。当应用程序使用数据并且缓冲区空间可用时，端点发送MAX_STREAM_DATA帧以允许对等方发送更多数据。

当接收到具有FIN位的STREAM帧时，流的最终大小是已知的（参见第4.4节）。然后，流的接收部分进入`Size Known`状态。在此状态下，端点不再需要发送MAX_STREAM_DATA帧，它只接收流数据的任何重传。

一旦接收到流的所有数据，接收部分就进入`Data Recvd`状态。这可能是由于接收到导致转换为`Size Known`的相同STREAM帧而发生的。在此状态下，端点具有所有流数据。可以丢弃它为流接收的任何STREAM或STREAM_DATA_BLOCKED帧。

`Data Recvd`状态持续存在，直到流数据已传送到应用程序。一旦传送了流数据，流就进入`Data Read`状态，这是一种终结状态。

在`Recv`或`Size Known`状态下接收到RESET_STREAM帧会使流进入`Reset Recvd`状态。这可能导致流数据传递到应用程序中断。

当流处于`Data Recvd`状态接收到RESET_STREAM时，端点可能接收到所有流数据。类似地，在接收RESET_STREAM帧之后剩余的流数据可能到达（`Reset Recvd`状态）。一个实现可以自由选择管理这种情况。发送RESET_STREAM意味着端点无法保证流数据的传递。但是，如果收到RESET_STREAM，则不要求不传送流数据。实现可以中断流数据的传送，丢弃未使用的任何数据，并立即发出RESET_STREAM的接收信号。或者，如果流数据被完全接收并且缓冲以供应用程序读取，则可以抑制或保留RESET_STREAM信号。在后一种情况下，流的接收部分从`Reset Recvd`转换为`Data Recvd`。

一旦应用程序已经被传送指示流被重置的信号，流的接收部分就转换到`Reset Recvd`状态，这是终结状态。

#### 3.3 允许的帧类型

流的发送方只发送三种帧类型，这些类型会影响发送方或接收方的流状态：STREAM（第19.8节），STREAM_DATA_BLOCKED（第19.13节）和RESET_STREAM（第19.4节）。

发送方不得从终结状态（`Data Recvd`或`Reset Recvd`）发送任何这些帧。发送者在发送RESET_STREAM后不得发送STREAM或STREAM_DATA_BLOCKED。也就是说，在终结状态和`Reset Sent`状态下。接收器可以在任何状态下接收这三个帧中的任何一个，因为可能延迟传送携带它们的分组。

流的接收器发送MAX_STREAM_DATA（第19.10节）和STOP_SENDING帧（第19.5节）。

接收器仅发送处于`Recv`状态的MAX_STREAM_DATA。接收器可以在没有收到RESET_STREAM帧的任何状态下发送STOP_SENDING;这是除`Reset Recvd`或`Reset Read`以外的状态。但是，在`Data Recvd`状态下发送STOP_SENDING帧几乎没有价值，因为已收到所有流数据。由于数据包的延迟传送，发送方可以在任何状态下接收这两个帧中的任何一个。

#### 3.4 双向流状态

双向流由发送和接收部分组成。实现可以将双向流的状态表示为发送和接收流状态的组合。当发送或接收部分处于非终结状态时，最简单的模型将流呈现为`open`，而当发送和接收流都处于终结状态时，流呈现为`closed`。

表2显示了与HTTP/2 中的流状态松散对应的双向流状态的复杂映射。这表明发送或接收部分流的多个状态被映射到相同的复合状态。请注意，这只是这种映射的一种可能性;此映射要求在转换到`closed`或`half-closed`状态之前确认数据。

| 发送部分               | 接收部分               | 复合状态             |
| ---------------------- | ---------------------- | -------------------- |
| No Stream/Ready        | No Stream/Recv *1      | idle                 |
| Ready/Send/Data Sent   | Recv/Size Known        | open                 |
| Ready/Send/Data Sent   | Data Recvd/Data Read   | half-closed (remote) |
| Ready/Send/Data Sent   | Reset Recvd/Reset Read | half-closed (remote) |
| Data Recvd             | Recv/Size Known        | half-closed (local)  |
| Reset Sent/Reset Recvd | Recv/Size Known        | half-closed (local)  |
| Reset Sent/Reset Recvd | Data Recvd/Data Read   | closed               |
| Reset Sent/Reset Recvd | Reset Recvd/Reset Read | closed               |
| Data Recvd             | Data Recvd/Data Read   | closed               |
| Data Recvd             | Reset Recvd/Reset Read | closed               |

> 备注 (*1)：如果尚未创建流，则该流被认为是`idle`，或者如果流的接收部分处于“Recv”状态而尚未接收到任何帧，则该流被认为是`idle`。

#### 3.5 询问状态过渡

如果端点不再对它在流上接收的数据感兴趣，它可以发送一个STOP_SENDING帧来标识该流，以提示在相反方向关闭流。这通常表示接收应用程序不再读取它从流中接收的数据，但不保证将忽略传入的数据。

发送STOP_SENDING后收到的STREAM帧仍计入连接和流量控制，即使这些帧在接收时将被丢弃。

STOP_SENDING帧请求接收端点发送RESET_STREAM帧。如果流处于就绪或发送状态，则接收STOP_SENDING帧的端点必须发送RESET_STREAM帧。如果流处于数据发送状态并且任何未完成的数据被声明丢失，则端点应该发送RESET_STREAM帧代替重传。

端点应该将错误代码从STOP_SENDING帧复制到它发送的RESET_STREAM帧，但是可以使用任何应用程序错误代码。发送STOP_SENDING帧的端点可以忽略它接收的任何RESET_STREAM帧中携带的错误代码。

如果在已经处于`Data Sent`状态的流上接收到STOP_SENDING帧，则希望停止在该流上重传先前发送的STREAM帧的端点必须首先发送RESET_STREAM帧。

STOP_SENDING应该仅针对未被对等方重置的流发送。 STOP_SENDING对于`Recv`或`Size Known`状态的流最有用。

如果包含先前STOP_SENDING的数据包丢失，则端点应发送另一个STOP_SENDING帧。但是，一旦为流接收到所有流数据或RESET_STREAM帧（即，流处于`Recv`或`Size Known`以外的任何状态）发送STOP_SENDING帧是不必要的。

希望终止双向流的两个方向的端点可以通过发送RESET_STREAM来终止一个方向，并且它可以通过发送STOP_SENDING帧来鼓励在相反方向上的快速终止。