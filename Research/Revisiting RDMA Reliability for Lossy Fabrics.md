## 问题
1. 为什么PFC死锁、阻塞问题会导致部署规模受到限制
## 研究背景

## Lossless Control Plane

![[DCP packet header.png]]
 57 bytes = 14 bytes MAC header + 20 bytes IP header + 8 bytes UDP header + 12bytes BTH header + 3 bytes for the MSN field.

1. BTH(Base Transport Header):
	 位置：标准 RDMA BTH 中已有 PSN 字段（论文继续使用），HO packet 中携带丢失包的 PSN 以便发送端精确重传。触发：在 HO packet 的 header 中必须包含目标丢包的 PSN。
2. MSN(Message Sequence Number):
	 作用：标识 message（上层请求）的序号（用来做 message-level 完成语义），论文把 MSN 放在 header（部分位于 RETH / BTH 扩展字段中）。在 ACK 中返回 eMSN（expected/ended MSN）以告诉发送端哪些 message 已完成。
3. SSN(Send Squence Number): 
	用于 Send / Write-with-Immediate 场景，帮助接收端在 out-of-order（OOO）情况下匹配正确的 Receive WQE，避免在中间包先到达时找不到目标缓冲区的问题。
4. RETH(Remote Extended Transport Header):
	问题：标准 RDMA 只在第一个 packet 包含 RETH（remote memory address）；这会在 OOO 到达时使中间包无法写入正确远端地址。
	解决：DCP 把 RETH 放到 **每一个** Write 的 packet 中（包括中间包），使得任何到达的包都能直接写入远端内存位置（order-tolerant）。这样移除了大额 reorder buffer 的需要。
5. sRetryNo（sender Retry Number）:
	 作用：当发送端触发 coarse-grained timeout 并重传整条 message 时，为了让接收端区分这是 timeout 重传还是第一次发送，数据包携带 sRetryNo，接收端据此决定是否计入 message 的 packet count（避免重复计数导致错误）。
### 效果
- **控制平面小而可靠**：header 仅 57 bytes，被放到 control queue 优先调度，几乎无丢失（设计目标）——从而 HO 通知可靠到达。
- **快速精确重传**：HO packet 带 PSN → 发送端立即精确重传单个丢失包（无需 bitmap 或 RTO 触发）。
- **摆脱 bitmap**：因“exactly-once”语义与 message-level 计数，接收端用 message-level counters 替代 per-packet bitmap，大幅减少 RNIC SRAM 需求（例：每 QP 只需几十字节）。
- **兼容 LB（load balancing）和 ECMP**：因为 HO-based 重传通过 header 通信且不依赖路径内状态，DCP 与 packet-level LB（ECMP）兼容。

## Efficient HO-based Retransmission

**核心思想：** 把“重传”能力放进 RNIC 硬件，且把“控制信息（header）”做成交换机优先保证的无损控制通道（control plane），当交换机拥塞时只丢 payload、保留 header（HO packet），接收端把 header 变成对发送端的重传请求，发送端在硬件内精确且快速重传指定包——这一切都在硬件内完成，避免等待 RTO 和复杂 bitmap。

![[arch.png]]

| 模块      | 全称                       | 作用                         |
| ------- | ------------------------ | -------------------------- |
| **QPC** | Queue Pair Context       | 记录通信双方状态（包括连接、序号、ACK等）     |
| **SQ**  | Send Queue               | 存放要发出去的任务（WQE）             |
| **RQ**  | Receive Queue            | 存放等待接收的缓冲区                 |
| **CQ**  | Completion Queue         | 存放完成通知（CQE）                |
| **MTT** | Memory Translation Table | 类似页表                       |
| **CC**  | Congestion Control       | 读队列延迟/trim率/RTT 等指标来调节发送速率 |

## Order-tolerant Packet Reception
![[Pasted image 20251013170634.png]]

## Bitmap-free Packet Tracking
通常使用bitsmap，但是消耗内存、CPU资源