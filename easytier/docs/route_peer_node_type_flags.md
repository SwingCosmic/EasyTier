# RoutePeerInfo 节点能力标记设计草案

## 背景与现状

`RoutePeerInfo` 已经承担 EasyTier 控制面中 peer 路由信息传播的职责。随着公开服务器列表、GUI、运维监控和生态扩展工具变多，上层组件经常需要快速判断一个 peer 是否具备某些基础设施能力，例如：

- 是否是 public server
- 是否支持 UDP/TCP 打洞辅助
- 是否可以承担数据中转或 peer RPC 中转
- 是否暴露 VPN portal、ACL、MagicDNS、exit node 等功能
- 是否支持 TCP/UDP/WS/KCP/QUIC、UPnP、SOCKS5、Windows UDP 广播或内核转发等通信方式
- 是否携带某个扩展应用定义的轻量发现标记

当前这些信息往往只能通过配置约定、hostname、公开服务器列表、运行时探测或业务侧 HTTP 扫描间接获得。该方式存在几个问题：

- 上层工具需要猜测 peer 的用途，容易出现误判。
- 公开服务器收集站点和 GUI 很难用统一字段展示能力矩阵。
- 扩展应用如果都直接占用全局 bit，后续很快会出现冲突。
- 用 hostname、group、feature flag 组合表达能力会让语义变得不稳定。

本文建议在 `RoutePeerInfo` 中增加通用的节点能力标记字段。该字段只表达粗粒度能力提示，不改变路由计算、连接建立、认证授权和数据转发语义。

## 设计目标

- 为 EasyTier 官方基础设施能力提供统一的轻量 bitset。
- 优先覆盖公开服务器列表和 GUI 最关注的能力：打洞、中转、portal、ACL，以及协议与通信方式矩阵。
- 为生态扩展应用提供可隔离的高位 bit 命名空间，避免互相抢占全局 bit。
- 复用现有 `RoutePeerInfo` 传播链路，不引入新的发现协议。
- 保持字段语义稳定，避免把端口、region、priority、URL、健康状态等易变结构化信息塞进 bitset。
- 保持向后兼容，老版本 peer 不携带字段时按未知能力处理。

## 非目标

- 不把该字段作为安全授权依据。
- 不替代 `PeerFeatureFlag` 已有的网络行为开关。
- 不在 EasyTier 核心中理解 EtDiscovery、EasytierGame、MCTier 等应用的业务语义。
- 不通过 bitset 表达完整节点状态、链路质量、服务端口、服务目录或租约信息。

## 字段定义

建议在 `peer_rpc.proto` 的 `RoutePeerInfo` 中增加三个字段：

```proto
message RoutePeerInfo {
  ...

  // Generic peer capability bitset.
  //
  // bits 0..15 are reserved for EasyTier-defined capabilities.
  // bits 16..31 are interpreted by node_type_app_id.
  uint32 node_type_flags = 25;

  // Listener / transport protocol bitset for this peer.
  //
  // This field only describes which underlay listener protocols or
  // transport entry styles the peer supports, such as tcp/udp/ws/wss/wg/quic.
  uint32 listener_protocols = 26;

  // Application namespace for node_type_flags bits 16..31.
  //
  // 0 means no application-specific flags.
  optional uint32 node_type_app_id = 27;
}
```

约束：

- `node_type_flags == 0` 且 `node_type_app_id == 0` 表示未声明能力或普通 peer。
- 一个 peer 可以同时设置多个 EasyTier 官方低位能力。
- `node_type_flags` 的低 16 位由 EasyTier 官方定义。
- `node_type_flags` 的高 16 位由 `node_type_app_id` 指定的扩展应用解释。
- `listener_protocols` 独立表达监听器协议或入口协议，不占用 `node_type_flags` 的官方低位空间。
- `node_type_app_id == 0` 时，消费者必须忽略高 16 位。
- 不认识某个 `node_type_app_id` 的消费者仍然可以展示低 16 位官方能力。

## 与 PeerFeatureFlag 的关系

`PeerFeatureFlag` 继续表达 EasyTier 内部已经使用的网络行为、协议能力或连接策略。`node_type_flags` 只面向路由元数据消费者，用于快速发现和展示 peer 具备的通用能力；`listener_protocols` 则专门表达监听器/入口协议能力。

两者允许存在少量冗余，但语义不同：

- `PeerFeatureFlag` 更偏运行时行为开关，可能直接影响连接、转发或协议处理。
- `node_type_flags` 更偏能力声明和发现提示，用于过滤候选 peer 或展示能力矩阵。
- `listener_protocols` 更偏“这个节点监听或暴露了哪些入口协议”，便于 GUI、CLI 和候选过滤直接使用。

例如 `is_public_server` 可以继续保留在 `PeerFeatureFlag` 中，同时 `node_type_flags.public_server` 让 GUI、CLI 和公开服务器列表更容易识别该能力。

## EasyTier 官方能力区

低 16 位预留给 EasyTier 官方能力。这里建议只保留“能力、行为、服务类型”类 bit，不再把监听器协议本身塞进 `node_type_flags`。

原因是 EasyTier 原生支持的监听协议更多，来自 `listeners`、`vpn_portal`、`config_server` 等配置的组合也更复杂，例如：

- `tcp`
- `udp`
- `ring`
- `wg`
- `ws`
- `wss`
- `quic`
- `faketcp`

如果继续把监听协议直接塞进低 16 位，很快就会挤占业务能力空间。因此更合适的做法是：

- `node_type_flags`
  - 表达“能做什么”
- `listener_protocols`
  - 表达“用什么协议监听/接入”

`quic` 比较特殊：

- 一方面，它是监听/接入协议的一种，应该出现在 `listener_protocols`
- 另一方面，像 `enable_quic_proxy`、`disable_quic_input`、`disable_relay_quic` 这类能力又说明它也是一种代理/封包方式

因此 `quic` 可以同时在两边出现：

- 在 `listener_protocols` 里表示“节点支持 QUIC listener / entry”
- 在 `node_type_flags` 里通过 `quic_proxy`、`relay_quic` 这类能力表达“节点支持以 QUIC 方式代理或转发”

| Bit | Hex | 名称 | 分组 | 含义 |
| ---: | ---: | --- | --- | --- |
| 0 | `0x0000_0001` | `public_server` | 节点角色提示 | 公开共享节点或 bootstrap 能力节点。 |
| 1 | `0x0000_0002` | `hole_punch_assist` | 连接建立能力 | 可参与打洞辅助。 |
| 2 | `0x0000_0004` | `relay_transport` | 转发能力 | 可为其他 peer 中转数据流量。 |
| 3 | `0x0000_0008` | `relay_peer_rpc` | 转发能力 | 可为其他 peer 中转 peer RPC / 控制流量。 |
| 4 | `0x0000_0010` | `vpn_portal` | 入口服务能力 | 暴露 VPN / portal 类入口能力。 |
| 5 | `0x0000_0020` | `socks5_portal` | 入口服务能力 | 暴露 SOCKS5 服务入口，允许客户端访问虚拟网络。 |
| 6 | `0x0000_0040` | `magic_dns` | 控制能力 | 提供或参与 MagicDNS 相关能力。 |
| 7 | `0x0000_0080` | `acl_enabled` | 控制能力 | 启用 ACL 或 credential group 访问控制能力。 |
| 8 | `0x0000_0100` | `exit_node` | 网络出口能力 | 可作为 exit node。 |
| 9 | `0x0000_0200` | `upnp_mapping` | 连接建立能力 | 支持运行时 UPnP/NAT-PMP 端口映射。 |
| 10 | `0x0000_0400` | `udp_broadcast_relay` | 特殊转发能力 | 支持 Windows UDP 广播捕获与转发。 |
| 11 | `0x0000_0800` | `kernel_forward` | 转发实现能力 | 支持通过系统内核转发流量。 |
| 12 | `0x0000_1000` | `kcp_proxy` | 代理/封包能力 | 支持用 KCP 代理 TCP 流。 |
| 13 | `0x0000_2000` | `quic_proxy` | 代理/封包能力 | 支持用 QUIC 代理 TCP 流。 |
| 14 | `0x0000_4000` | `relay_foreign_network` | 共享网络能力 | 支持为 foreign network 中继流量。 |
| 15 | `0x0000_8000` | `shared_relay` | 共享网络能力 | 支持作为共享节点为其他网络或 peer 提供 relay。 |

组合解释示例：

- `hole_punch_assist + listener_protocols.tcp + listener_protocols.udp`
  - 表示节点既支持 TCP 打洞辅助，也支持 UDP 打洞辅助
- `relay_transport + relay_peer_rpc + listener_protocols.ws + listener_protocols.wss`
  - 表示节点既能中转数据，也能中转 peer RPC，并支持 WS/WSS 入口
- `vpn_portal + listener_protocols.wg`
  - 表示节点提供 WireGuard 风格 VPN portal
- `socks5_portal`
  - 表示节点提供 SOCKS5 入口服务，本身不依赖额外 listener protocol bit 组合解释
- `kcp_proxy + listener_protocols.udp`
  - 表示节点支持以 UDP 相关入口承载 KCP 代理能力
- `quic_proxy + listener_protocols.quic`
  - 表示节点既支持 QUIC listener，也支持用 QUIC 代理 TCP 流
- `upnp_mapping + hole_punch_assist`
  - 表示节点支持通过 UPnP/NAT-PMP 改善连接建立
- `kernel_forward + relay_transport`
  - 表示节点支持通过系统内核路径进行中转

如果某项能力与监听协议无关，例如 `acl_enabled`、`magic_dns`、`exit_node`，则不需要再额外组合 `listener_protocols`。

## 监听协议字段建议

建议新增 `listener_protocols` 字段，专门存放“监听方式协议”。

参考 `easytier/locales/app.yml` 中 `listeners`、`config_server`、`vpn_portal` 的现状，首批建议覆盖：

| Bit | Hex | 名称 | 含义 |
| ---: | ---: | --- | --- |
| 0 | `0x0000_0001` | `tcp` | 支持 TCP listener / entry |
| 1 | `0x0000_0002` | `udp` | 支持 UDP listener / entry |
| 2 | `0x0000_0004` | `ring` | 支持 ring listener |
| 3 | `0x0000_0008` | `wg` | 支持 WireGuard listener / portal |
| 4 | `0x0000_0010` | `ws` | 支持 WebSocket listener |
| 5 | `0x0000_0020` | `wss` | 支持 secure WebSocket listener |
| 6 | `0x0000_0040` | `quic` | 支持 QUIC listener |
| 7 | `0x0000_0080` | `faketcp` | 支持 FakeTCP listener |
| 8..31 | reserved | 预留 | 预留给后续 listener 协议 |

这个字段的语义应尽量保持简单：

- 只表示“是否支持/暴露这种 listener 协议”
- 不表达该协议是否被用于打洞、relay、proxy 或 portal
- 更细粒度的行为仍由 `node_type_flags` 的能力位来表达

以上仅为示例说明，不代表具体实现定义。实现时应尽量只设置可以从本地配置或运行时状态稳定推导的 bit。暂时无法精确定义来源的能力可以先预留，不必在首版强行填充。

这样做的直接好处是：

- `node_type_flags` 不会被大量 listener 协议名迅速耗尽。
- 新增 listener 协议时只需要扩展 `listener_protocols`，不需要重做官方能力表。
- GUI 和公开服务器列表可以同时展示“能力矩阵”和“监听协议矩阵”。
- 过滤候选 peer 时，可以更容易表达“需要 relay，并且必须支持 WS/WSS listener”或“需要 portal，并且必须支持 WG listener”这类组合条件。

## 应用扩展区

高 16 位不做全局静态均分，而是由 `node_type_app_id` 选择解释命名空间。

| App ID | 名称 | 建议用途 |
| ---: | --- | --- |
| 0 | none | 不包含应用私有高位标记。 |
| 1 | EasyTierDiscovery | 服务注册、服务发现和 registry bootstrap。 |
| 2 | EasytierGame | 基于 EasyTier 的游戏联机启动器和房间辅助工具。 |
| 3..1023 | assigned/community | 可约定给常见生态应用，比如AstralGame、MCTier等。 |
| 1024..65535 | private/experimental | 私有部署、实验工具或临时约定。 |

规则：

- EasyTier 核心不需要理解高位 bit 的应用业务含义。
- 扩展应用只在自己的 `node_type_app_id` 命名空间内解释高位 bit。
- 公开服务器列表或 GUI 如果不认识某个 `node_type_app_id`，可以只展示官方低位能力。
- 首版仅承载一个主要应用命名空间。需要更多结构化信息时，应由应用自己的 API 或协议返回。

## 应用使用示例

以下示例只用于说明命名空间如何避免 bit 冲突，不要求 EasyTier 核心理解这些业务含义。

EasyTierDiscovery 可以使用 `node_type_app_id = 1`，并将 bit 16/17/18 分别解释为 `registry`/`worker`/`client`。当高位角色 bit 全 0 时，表示该 peer 未声明任何 EtDiscovery 业务角色，可视为 `empty` 或普通未命名节点。若后续需要显式表达 `observer`、`gateway`、`relay` 等额外职责，应分配新的 role bit，而不是继续让 `empty` 承载语义。worker/client 从 `RoutePeerInfo` 中筛选出候选 registry 后，再访问 advertise 端点获取协议版本、network、region、priority、capability 等明细。

EasytierGame（[EasyTier/EasytierGame](https://github.com/EasyTier/EasytierGame)）可以使用 `node_type_app_id = 2`，在高位标记启动器节点、房间主机、自建服务器入口、配置分享辅助或 UDP broadcast relay 等游戏联机场景能力。


游戏名、房间名、端口、服务地址、租约、健康状态等信息仍应留在应用自己的协议中，不进入 `RoutePeerInfo` 的 bitset。

## 推荐常量

```rust
pub mod node_type_flags {
    pub const PUBLIC_SERVER: u32 = 1 << 0;
    pub const HOLE_PUNCH_ASSIST: u32 = 1 << 1;
    pub const RELAY_TRANSPORT: u32 = 1 << 2;
    pub const RELAY_PEER_RPC: u32 = 1 << 3;
    pub const VPN_PORTAL: u32 = 1 << 4;
    pub const SOCKS5_PORTAL: u32 = 1 << 5;
    pub const MAGIC_DNS: u32 = 1 << 6;
    pub const ACL_ENABLED: u32 = 1 << 7;
    pub const EXIT_NODE: u32 = 1 << 8;
    pub const UPNP_MAPPING: u32 = 1 << 9;
    pub const UDP_BROADCAST_RELAY: u32 = 1 << 10;
    pub const KERNEL_FORWARD: u32 = 1 << 11;
    pub const KCP_PROXY: u32 = 1 << 12;
    pub const QUIC_PROXY: u32 = 1 << 13;
    pub const RELAY_FOREIGN_NETWORK: u32 = 1 << 14;
    pub const SHARED_RELAY: u32 = 1 << 15;

    pub const APP_BIT_0: u32 = 1 << 16;
    pub const APP_BIT_1: u32 = 1 << 17;
    pub const APP_BIT_2: u32 = 1 << 18;
    pub const APP_BIT_3: u32 = 1 << 19;
    pub const APP_BIT_4: u32 = 1 << 20;
    pub const APP_BIT_5: u32 = 1 << 21;
    pub const APP_BIT_6: u32 = 1 << 22;
    pub const APP_BIT_7: u32 = 1 << 23;
    pub const APP_BIT_8: u32 = 1 << 24;
    pub const APP_BIT_9: u32 = 1 << 25;
    pub const APP_BIT_10: u32 = 1 << 26;
    pub const APP_BIT_11: u32 = 1 << 27;
    pub const APP_BIT_12: u32 = 1 << 28;
    pub const APP_BIT_13: u32 = 1 << 29;
    pub const APP_BIT_14: u32 = 1 << 30;
    pub const APP_BIT_15: u32 = 1 << 31;
}

pub mod listener_protocols {
    pub const TCP: u32 = 1 << 0;
    pub const UDP: u32 = 1 << 1;
    pub const RING: u32 = 1 << 2;
    pub const WG: u32 = 1 << 3;
    pub const WS: u32 = 1 << 4;
    pub const WSS: u32 = 1 << 5;
    pub const QUIC: u32 = 1 << 6;
    pub const FAKETCP: u32 = 1 << 7;
}

pub mod node_type_app_id {
    pub const NONE: u32 = 0;
    pub const ETDISCOVERY: u32 = 1;
    pub const EASYTIER_GAME: u32 = 2;
    pub const MCTIER: u32 = 3;
}
```

## 兼容性说明

- protobuf 新字段对旧版本 peer 保持兼容，旧版本不发送时按 `0` 处理。
- 新版本消费者必须把缺失字段视为“未知能力”，不能视为错误。
- 未识别的 `node_type_app_id` 不应影响 route metadata 解析。
- 不支持该字段的部署仍可继续使用显式配置、公开服务器列表或本地缓存作为 fallback。

## 渐进式落地计划

### 阶段 1：字段与常量

- 在 `RoutePeerInfo` 中增加 `node_type_flags` 和 `node_type_app_id`。
- 在 `RoutePeerInfo` 中增加 `listener_protocols` 字段。
- 增加 EasyTier 官方低位能力常量。
- 增加 listener protocol 常量。
- 增加常见应用命名空间 ID 常量。

### 阶段 2：官方能力填充

- 从本地配置和运行时状态填充首批稳定能力 bit。
- 优先填充 public server、打洞、中转、peer RPC relay、portal、SOCKS5、ACL、MagicDNS、exit node、UPnP、UDP 广播中继、KCP/QUIC proxy、kernel forward 等明确能力。
- 从 `listeners`、`vpn_portal`、`config_server` 和相关入口配置填充 `listener_protocols`。
- 对推导条件不稳定的 bit 暂缓填充。

### 阶段 3：观测与展示

- 在 CLI / API 输出中暴露字段，便于调试和外部工具消费。
- GUI 和公开服务器列表可以基于低位官方能力展示能力矩阵。
- 未识别的高位应用标记保持透传或忽略。

### 阶段 4：扩展应用接入

- 为扩展应用提供受控入口设置 `node_type_app_id` 和高位 bit。
- 应用通过自己的 well-known endpoint、HTTP API 或其他协议返回结构化明细。
- EasyTier 核心只负责传播命名空间和 bitset，不内置应用业务模型。

## 注意事项

- 本字段是能力提示，不是认证结果。
- ACL、credential、shared node 等安全相关能力不能只依赖 bit 判断。
- 应用高位 bit 不应反向影响 EasyTier 路由计算。
- 如果后续生态应用数量明显增加，可以再引入正式的 `node_type_app_id` 分配登记规则。
