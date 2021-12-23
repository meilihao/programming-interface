# iscsi
## 概念
ref:
- [使用SPDK部署iSCSI](https://pancrasl.gitee.io/2021/03/04/spdk-iscsi/)

概念:
- Network Portal

    网络端口. 网络实体的一个组成部分，它有一个 TCP/IP 地址。 网络端口在 Initiator 用 IP 地址标识， 在 Target 用 IP 地址＋侦听的 TCP 端口标识。

- Session

    连接 Initiator 和 Target 的一组 TCP 连接构成一个 session(可以简单理解为 I_T nexus)。可以向 session 添加 TCP 连接，也可以把 TCP 连接从 session 删除。 也就是说一个session中是可以有多个连接的。通过一个 session 的所有连接，Initiator 只看到同一个 Target。

- Connection

    一个 TCP 连接。Initiator 和 Target 之间使用一或者多个 TCP 连接通信。

- CID(Connection ID)

    一个 session 里的每个 connection 用 CID 进行标识，该标识在 session 范围内是唯一。CID 由 Initiator 产生，在 login 请求和使用 logout 关闭 连接时传递给 Target。

- SSID（Session ID）

    一个 iSCSI Initiator 与 iSCSI Target 之间的会话（Session）由会话ID（SSID）定义，该会话ID是一个由发起方部分（ISID）和目标部分（Target Portal Group Tag）组成的元组。 ISID 在会话建立时由发起者明确指定。 Target Portal Group Tag 由发起者在连接建立时选择的 TCP端口来隐式指定。 当给定 targetName 时，targetPortalGroupTag 也必须由目标在连接建立期间作为确认返回。

- Portal Groups

    网络端口组。iSCSI session 支持多连接，一些实现能把通过多个端口建立的多个连接捆绑到一个 session。 一个 iSCSI 网络实体的多个网络端口被定义为一个网络端口组，把该组和一个 session 联系起来，该 session 就可以捆绑通过该组内多个端口建立的多个连接，再使它们一起协同工作以达到捆绑的目的。每一个该组的 session 并不需要包括该组的所有网络端口。一个 iSCSI 节点可能有一或者多个网络端口组，但是每一个 iSCSI 使用的网络端口只能属于 iSCSI 节点的一个组。

- Target Portal Group Tag

    网络端口组标识。使用 16 比特的数标识一个网络端口组。在 一个 iSCSI 节点里，所有具有同样组标志的端口构成一个网络端口组。

- iSCSI Task

    一个 iSCSI 任务是指一个需要响应的 iSCSI 请求。

- I_T nexus

    I_T nexus 是指一个 SCSI Initiator 的端口和一个 SCSI Target 端口之间 的关系。 对于 iSCSI， 这个关系对应一个 session， 它指 session 的 Initiator 端和 iSCSI Target 网络端口组之间的关系。I_T nexus 的标识是一对端口名称(iSCSI Initiator 名称＋i＋ISID，iSCSI Target 名称＋t＋网络端口组标识)。 PDU (Protocol Data Unit): Initiator 和 Target 之间通信时把信息分割为消息。这些 消息称为 iSCSI PDU。 SSID (Session ID): iSCSI Initiator 和 iSCSI Target 之间的 session 用 SSID 进行标识， 该标识由 Initiator 部分的 ISID 和 Target 部分的 TPGT 构成。

- ISID（The Initiator part of the Session Identifier

    发起方会话标识，由 Initiator 在 session 建立的时候明确给出，

- TSIH (Target Session Identifying Handle)

    Target 分配给与特定名称 Initiator 建立的 session 的标识。 但是 0 值被保留着用于 Initiator 告知 Target 这是一个新 session。 在为一个 session 添加一个 connect 时，TSIH 已经隐含指明