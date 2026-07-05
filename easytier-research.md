# EasyTier 仓库调研报告

## 1. 项目结构与子项目关系

### 1.1 Workspace 总览
根工作区通过 [`Cargo.toml`](Cargo.toml) 的 `[workspace]` 组织多个 Rust 子项目，`members` 列出所有成员，默认构建成员为 `"easytier"` 与 `"easytier-web"`，`exclude` 排除需要 OHOS SDK 的 `easytier-contrib/easytier-ohrs`。

当前核心工作区成员：

- [`easytier`](easytier/Cargo.toml)：核心库与两个命令行二进制 `easytier-core`、`easytier-cli`。
- [`easytier-gui/src-tauri`](easytier-gui/src-tauri/Cargo.toml)：Tauri GUI 后端，依赖核心库 `easytier`。
- [`easytier-web`](easytier-web/Cargo.toml)：配置服务器 / Web 控制台后端，依赖核心库 `easytier`。
- [`easytier-contrib/easytier-ffi`](easytier-contrib/easytier-ffi/Cargo.toml)：核心能力的 C ABI 封装，输出 `cdylib`。
- [`easytier-contrib/easytier-android-jni`](easytier-contrib/easytier-android-jni/Cargo.toml)：Android JNI 绑定层，输出 `cdylib`，依赖 `easytier` 与 `easytier-ffi`。
- [`easytier-contrib/easytier-uptime`](easytier-contrib/easytier-uptime/Cargo.toml)：节点健康检查 / 在线监控服务。

### 1.2 顶层目录功能

- [`easytier/`](easytier/Cargo.toml)：项目核心实现，包含协议、隧道、路由、实例生命周期、RPC、CLI。
- [`easytier/docs/`](easytier/docs/credential_peer.md)：核心设计草案与架构文档，质量高于根 README，尤其适合理解 secure mode、relay、credential。
- [`easytier-gui/`](easytier-gui/src-tauri/Cargo.toml)：桌面 GUI（前端 + Tauri Rust 后端）。
- [`easytier-web/`](easytier-web/Cargo.toml)：配置中心 / 控制台，既提供 Web API，也给 `easytier-core` 的远程配置客户端使用。
- [`easytier-contrib/`](easytier-contrib/easytier-ffi/Cargo.toml)：外围集成物，包括 FFI、Android JNI、HarmonyOS、Uptime、Magisk。
- [`script/`](script/install.sh)：安装与测试脚本。
- [`assets/`](assets/config-page.png)：README 与站点素材。
- 根 [`README_CN.md`](README_CN.md)：用户向导、安装与高层使用说明。

### 1.3 `easytier` 核心 crate 内部模块关系
核心模块出口声明在 [`easytier/src/lib.rs`](easytier/src/lib.rs)：

- `core`：`easytier-core` 命令行入口逻辑。
- `instance_manager`：多实例生命周期编排。
- `instance`：单实例运行时，含监听器、虚拟网卡、IPv6 公网提供器、DNS。
- `peers`：Peer 连接、路由、转发、secure session、relay、foreign network。
- `connector`：直连、手工连接、DNS/HTTP 发现、打洞。
- `tunnel`：TCP / UDP / WebSocket / WireGuard / QUIC / fake TCP 等底层隧道。
- `gateway`：TCP/UDP/ICMP/KCP 代理、用户态协议栈。
- `rpc_service`：本地管理 RPC 服务。
- `web_client`：连接远端配置中心。
- `service_manager`：系统服务安装/管理。
- `proto`：protobuf 与 RPC 类型。
- `common`：配置、常量、全局上下文、网卡配置、命名空间等基础设施。

### 1.4 子项目依赖链

#### 核心链路
- `easytier-core` 只是薄启动器，真正执行 [`core::main()`](easytier/src/core.rs:1564)。
- `core::main()` 解析配置、创建 [`NetworkInstanceManager::new()`](easytier/src/instance_manager.rs:47)、启动 RPC 门户与网络实例。
- 每个实例由 [`NetworkInstanceManager::run_network_instance()`](easytier/src/instance_manager.rs:110) 创建 `NetworkInstance` 并启动。

#### 管理面链路
- `easytier-cli` 通过 `StandAloneClient` 调用本地 `rpc_service` 暴露的服务。
- `easytier-web` 作为远端配置中心；`easytier-core` 可通过 `web_client::run_web_client()` 连回该服务，按远程配置驱动实例。

#### 图形与移动集成链路
- `easytier-gui` 直接依赖 `easytier`，因此 GUI 后端可直接复用核心逻辑。
- `easytier-ffi` 提供稳定 C ABI 给 C / Go / C# 等语言。
- `easytier-android-jni` 再次包装 `easytier-ffi` 以适配 Android Java/Kotlin。

### 1.5 关键外部依赖及作用

#### 核心运行时
- `tokio`：异步运行时，整个实例与网络 IO 的基础。
- `clap`：`easytier-core` / `easytier-cli` 参数解析。
- `prost` 与 `prost-reflect`：protobuf / RPC 类型。

#### 网络与隧道
- `tun-easytier`：跨平台 TUN 设备。
- `quinn`：QUIC 隧道。
- `tokio-websockets`：WebSocket 隧道。
- `boringtun-easytier`：WireGuard 门户。
- `smoltcp`：用户态 TCP/IP 栈，用于 `--use-smoltcp` 场景。
- `stun_codec`：NAT 探测与打洞。

#### 安全与加密
- `snow`：Noise 握手。
- `x25519-dalek`：静态/临时密钥。
- `aes-gcm`、`ring`、`openssl`：数据面加密实现。

#### Web / 管理面
- `axum`：Web 控制台后端。
- `sea-orm`：数据库 ORM。
- `openidconnect`：OIDC 登录。

---

## 2. `easytier-core` 命令行工具：完整使用方法与参数

### 2.1 入口与职责
`easytier-core` 二进制由 `[[bin]] name = "easytier-core"` 定义，启动文件只调用 [`core::main()`](easytier/src/core.rs:1564)。完整参数定义位于以下结构体（均在同文件 `easytier/src/core.rs`）：

- [`struct Cli`](easytier/src/core.rs:88)：顶层 CLI 参数。
- [`struct NetworkOptions`](easytier/src/core.rs:144)：网络实例级参数。
- [`struct LoggingOptions`](easytier/src/core.rs:735)：日志参数。
- [`struct RpcPortalOptions`](easytier/src/core.rs:773)：RPC 门户参数。

### 2.2 基本使用模式

#### 模式 A：纯命令行启动单实例
适用于无配置文件快速组网。只要没有 `--config-file` / `--config-dir` / `--config-server`，并且不是 `--daemon`，就会在 `run_main()` 里构造一个临时 `TomlConfigLoader::default()`。

示例：
```bash
sudo easytier-core -i 10.144.144.2 -p udp://1.2.3.4:11010
```
对应参数映射：`-i/--ipv4`、`-p/--peers`。

#### 模式 B：通过 TOML 文件启动一个或多个实例
通过 `--config-file` 或 `--config-dir` 加载。每个 TOML 文件都会在 `load_config_from_file()` 后交给 `manager.run_network_instance()`。

#### 模式 C：远程配置客户端
通过 `--config-server` 进入 Web 配置托管模式，调用 `web_client::run_web_client()`。

### 2.3 通用顶层参数
以下参数定义在 [`struct Cli`](easytier/src/core.rs:88)：

- `-w, --config-server`：远程配置服务器地址；环境变量 `ET_CONFIG_SERVER`。
- `--machine-id`：指定机器标识；环境变量 `ET_MACHINE_ID`。
- `-c, --config-file`：一个或多个 TOML 文件，支持逗号分隔；环境变量 `ET_CONFIG_FILE`。
- `--config-dir`：扫描目录中的 `.toml` 配置；环境变量 `ET_CONFIG_DIR`。
- `--gen-autocomplete`：生成 shell 自动补全。
- `--check-config`：仅校验配置，不实际运行；实现见 `validate_config()`。
- `--daemon`：将实例注册为 daemon 管理模式。
- `--disable-env-parsing`：禁用从环境变量补全配置文件。

### 2.4 网络身份与地址参数
以下参数定义在 [`struct NetworkOptions`](easytier/src/core.rs:144)：

- `--network-name`：虚拟网络名；环境变量 `ET_NETWORK_NAME`。
- `--network-secret`：共享密钥；环境变量 `ET_NETWORK_SECRET`。
- `-i, --ipv4`：本节点虚拟 IPv4；环境变量 `ET_IPV4`。
- `--ipv6`：本节点虚拟 IPv6；环境变量 `ET_IPV6`。
- `--dhcp`：是否启用 DHCP 自动分配地址。
- `--hostname`：节点主机名。
- `-m, --instance-name`：实例名称。

### 2.5 连接、监听与拓扑参数
（同 [`struct NetworkOptions`](easytier/src/core.rs:144)）

- `-p, --peers`：初始 peer URI 列表，逗号分隔。
- `-e, --external-node`：附加外部节点 URI。
- `-n, --proxy-networks`：共享的子网 CIDR 列表。
- `-l, --listeners`：监听地址列表；解析逻辑见 `Cli::parse_listeners()`。
- `--mapped-listeners`：对外映射监听地址。
- `--no-listener`：禁止创建监听器。
- `--default-protocol`：默认协议。
- `--manual-routes`：手工路由 CIDR。
- `--relay-network-whitelist`：允许中继的网络白名单。

### 2.6 VPN / 代理 / 网关参数
（同 [`struct NetworkOptions`](easytier/src/core.rs:144)）

- `--vpn-portal`：启用 WireGuard 门户，格式如 `wg://0.0.0.0:11013/10.14.14.0/24`；解析逻辑见 `cfg.set_vpn_portal_config()`。
- `--port-forward`：端口转发规则，格式如 `udp://0.0.0.0:12345/10.126.126.1:12345`；构造逻辑见 `PortForwardConfig`。
- `--accept-dns`：接受 DNS 接管。
- `--tld-dns-zone`：顶级 Magic DNS 区域。
- `--private-mode`：启用私有模式。
- `--foreign-relay-bps-limit`：对 foreign network 中继限速。
- `--instance-recv-bps-limit`：实例接收限速。
- `--tcp-whitelist`：TCP 白名单端口集。
- `--udp-whitelist`：UDP 白名单端口集。

### 2.7 安全与认证参数
（同 [`struct NetworkOptions`](easytier/src/core.rs:144)）

- `-u, --disable-encryption`：关闭数据面加密。
- `--encryption-algorithm`：显式指定加密算法。
- `--secure-mode`：启用 secure mode（Noise）。
- `--local-private-key`：本地静态私钥。
- `--local-public-key`：本地静态公钥。
- `--credential`：临时凭据私钥，隐含 secure mode；实际处理见 `process_secure_mode_cfg()`。
- `--credential-file`：凭据文件路径。

### 2.8 P2P / NAT 穿透 / 中继参数
（同 [`struct NetworkOptions`](easytier/src/core.rs:144)）

- `--p2p-only`：仅允许 P2P。
- `--lazy-p2p`：按需建立 P2P。
- `--disable-p2p`：关闭 P2P。
- `--disable-udp-hole-punching`：禁用 UDP 打洞。
- `--disable-tcp-hole-punching`：禁用 TCP 打洞。
- `--disable-sym-hole-punching`：禁用对称 NAT 打洞。
- `--disable-upnp`：禁用 UPnP。
- `--relay-all-peer-rpc`：允许所有 peer RPC 经 relay。
- `--need-p2p`：要求尝试建立 P2P。
- `--disable-relay-kcp`：关闭 relay KCP。
- `--disable-relay-quic`：关闭 relay QUIC。
- `--enable-relay-foreign-network-kcp`：foreign network 开启 KCP 中继。
- `--enable-relay-foreign-network-quic`：foreign network 开启 QUIC 中继。
- `--stun-servers`：STUN 服务器列表。
- `--stun-servers-v6`：IPv6 STUN 服务器列表。

### 2.9 虚拟网卡、出口节点与性能参数
（同 [`struct NetworkOptions`](easytier/src/core.rs:144)）

- `--multi-thread`：启用多线程数据处理。
- `--multi-thread-count`：多线程数量。
- `--disable-ipv6`：禁用 IPv6。
- `--dev-name`：虚拟网卡设备名。
- `--mtu`：链路 MTU。
- `--latency-first`：路由优先低时延。
- `--exit-nodes`：出口节点地址列表。
- `--enable-exit-node`：允许本节点作为出口。
- `--proxy-forward-by-system`：使用系统协议栈转发代理流量。
- `--no-tun`：不创建 TUN。
- `--use-smoltcp`：启用 `smoltcp` 用户态协议栈。
- `--enable-udp-broadcast-relay`：开启 Windows UDP 广播转发。
- `--compression`：压缩算法，目前支持 `none` / `zstd`。
- `--bind-device`：绑定物理出接口。
- `--socket-mark`：仅 Linux / Android / Fuchsia 支持的 SO_MARK。
- `--enable-kcp-proxy`：启用 KCP 代理。
- `--disable-kcp-input`：关闭 KCP 入站。
- `--enable-quic-proxy`：启用 QUIC 代理。
- `--disable-quic-input`：关闭 QUIC 入站。

### 2.10 IPv6 公网地址提供器参数
（同 [`struct NetworkOptions`](easytier/src/core.rs:144)）

- `--ipv6-public-addr-provider`：是否启用公网 IPv6 提供器。
- `--ipv6-public-addr-auto`：自动探测公网 IPv6 前缀。
- `--ipv6-public-addr-prefix`：显式指定前缀。

### 2.11 日志与管理 RPC 参数
日志参数定义在 [`struct LoggingOptions`](easytier/src/core.rs:735)，RPC 参数定义在 [`struct RpcPortalOptions`](easytier/src/core.rs:773)：

- `--console-log-level`：控制台日志级别。
- `--file-log-level`：文件日志级别。
- `--file-log-dir`：文件日志目录。
- `--file-log-size`：单日志文件大小。
- `--file-log-count`：保留日志文件数。
- `-r, --rpc-portal`：本地 RPC 门户地址。
- `--rpc-portal-whitelist`：RPC 门户白名单。

### 2.12 推荐使用组合
常用场景参考 [`README_CN.md`](README_CN.md) 中"共享节点快速组网""去中心化组网""子网代理""WireGuard 集成"等章节。核心安装脚本位于 [`script/install.sh`](script/install.sh)。

---

## 3. `easytier` 核心库架构、运行模式与工作流程

### 3.1 架构总览
核心库导出模块在 [`easytier/src/lib.rs`](easytier/src/lib.rs)。按职责可分为：

- 配置与上下文层：`common`
- 实例编排层：`core`、`instance_manager`、`instance`
- 控制面：`peers`、`rpc_service`、`peer_center`
- 数据面：`tunnel`、`gateway`、`vpn_portal`
- 外部控制：`web_client`、`service_manager`

### 3.2 运行模式

#### 模式 1：本地 CLI 单实例 / 多实例模式
由 [`run_main()`](easytier/src/core.rs:1352) 解析 CLI / TOML 后调用 `NetworkInstanceManager::run_network_instance()`。

#### 模式 2：远程配置托管模式
带 `--config-server` 时启动 `web_client::run_web_client()`，通过配置中心动态创建 / 删除实例。

#### 模式 3：系统服务模式
Windows 下 `windows_service::service_dispatcher::start()` 尝试接入 SCM；CLI 还可通过 `ServiceSubCommand` 安装 / 启停系统服务。

#### 模式 4：FFI / JNI 嵌入模式
外部应用不运行 `easytier-core` 进程，而是直接通过 [`easytier-ffi`](easytier-contrib/easytier-ffi/src/lib.rs) 的 `parse_config()`、`run_network_instance()` 或 Android JNI 等接口嵌入运行时。

### 3.3 单实例核心组件

#### 全局上下文 + 配置
单实例以 [`GlobalCtx`](easytier/src/instance/instance.rs:24) 为共享状态中心，内部承载配置、事件总线、标志位、地址、网络命名空间等。

#### PeerManager
核心调度器是 [`struct PeerManager`](easytier/src/peers/peer_manager.rs:146)，内部关键字段：

- 直连 peer 表 `peers: Arc<PeerMap>`
- 路由实现 `route_algo_inst`
- foreign network 管理 `foreign_network_manager`
- relay 管理 `relay_peer_map`
- 数据面加密器 `encryptor`
- peer 级 secure session 存储 `peer_session_store`

#### 实例管理器
多实例总控是 [`struct NetworkInstanceManager`](easytier/src/instance_manager.rs:30)，维护活跃实例表、实例停止任务、错误收集、远程配置互斥锁等。

#### 虚拟网卡与路由同步
虚拟网卡上下文为 [`NicCtx`](easytier/src/instance/virtual_nic.rs:798)，负责：

- 创建 TUN / 接入移动端 TUN FD
- 启动 `NIC -> Peer` 与 `Peer -> NIC` 双向转发任务（`do_forward_nic_to_peers_task()` 与 `do_forward_peers_to_nic()`）
- 更新代理子网路由（`run_proxy_cidrs_route_updater()`）
- 更新公网 IPv6 地址 / 路由（`run_public_ipv6_route_updater()` 与 `run_public_ipv6_addr_updater()`）

### 3.4 secure mode / relay / credential 的设计文档结论

#### secure mode
设计文档 [`peer_conn_secure_mode_v3.md`](easytier/docs/peer_conn_secure_mode_v3.md) 说明：握手使用 Noise；数据面不直接使用 `snow::TransportState`，而是把 12B 明文 nonce 附在包尾，以支持乱序隧道；多条 `PeerConn` 共享一个 `PeerSession`。

#### relay
设计文档 [`relay_peer_manager_design.md`](easytier/docs/relay_peer_manager_design.md) 说明：顶层由 `PeerManager` 同时持有 `PeerMap` 与 `RelayPeerMap`；非直连发送流程由 `RelayPeerMap` 选下一跳并回调 `send_msg_directly`；secure mode 下 relay 会话需通过数据面握手消息建立（`RelayHandshake` / `RelayHandshakeAck`）。

#### credential
设计文档 [`credential_peer.md`](easytier/docs/credential_peer.md) 说明：凭据本质是 X25519 密钥对；管理节点生成并通过 OSPF 同步可信公钥列表（`trusted_credential_pubkeys`）；临时节点通过凭据私钥加入网络，不要求持有 `network_secret`。

### 3.5 大致工作流程

**步骤 1：程序入口与参数解析** — [`core::main()`](easytier/src/core.rs:1564) 初始化 locale、panic handler、profiling、Windows service 兼容逻辑，再通过 `parse_cli()` 获取参数。

**步骤 2：创建实例管理器与本地 RPC 门户** — `run_main()` 初始化日志后，创建 `NetworkInstanceManager`，然后建立 `ApiRpcServer::new(...).serve()`。这解释了为什么 `easytier-cli` 默认连接本地 `127.0.0.1:15888`。

**步骤 3：如有需要，启动远程配置客户端** — 若传入 `config_server`，则启动 `web_client::run_web_client()`，让远程 Web 平台驱动实例创建 / 更新 / 删除。

**步骤 4：加载并合并配置** — 配置来源可以是 CLI 直接参数、`--config-file`、`--config-dir` 或远程配置服务器。CLI 与文件配置的合并策略由 `NetworkOptions::can_merge()` 和 `NetworkOptions::merge_into()` 控制。

**步骤 5：启动实例** — 每份最终配置进入 `NetworkInstanceManager::run_network_instance()`。该函数创建 `NetworkInstance::new`，调用 `instance.start()`，并为其挂接停止观察任务。

**步骤 6：实例内部启动网络面** — 从 [`instance/instance.rs`](easytier/src/instance/instance.rs) 的依赖关系可见，单实例会组合：连接器（`DirectConnectorManager`、`TcpHolePunchConnector`、`UdpHolePunchConnector`）、Peer 管理（`PeerManager`）、各类代理（`TcpProxy`、`UdpProxy`、`IcmpProxy`）、监听器（`ListenerManager`）、VPN 门户（`VpnPortal`）。

**步骤 7：虚拟网卡与数据面收发** — [`NicCtx::run()`](easytier/src/instance/virtual_nic.rs:1337) 创建 TUN，建立流和 sink，然后从 NIC 读包后根据 IP 版本转发，从 peer 收到包后写回 NIC，初始化 IPv4 / IPv6 地址、代理路由、公网 IPv6 路由。

**步骤 8：控制面同步、选路与转发** — [`PeerManager::new()`](easytier/src/peers/peer_manager.rs:232) 创建 `PeerMap`、`PeerSessionStore`、`PeerRpcManager`、`PeerRoute`、`ForeignNetworkManager`、`ForeignNetworkClient` 等对象，承担 peer 建连与多连接复用、OSPF 风格路由同步、foreign network 中继、secure session 与 relay session 管理、按路由 / 延迟 / 出口节点策略转发数据。

### 3.6 平台差异

#### Windows
- 支持 Windows Service（`define_windows_service!` 与 `win_service_main()`）。
- TUN 创建前会尝试加入防火墙白名单（`add_self_to_firewall_allowlist()`）。
- 设备名可能自动生成（`random_dev_name`）。
- 会对网卡 GUID 做额外注册表与 NetBIOS 配置（`RegistryManager::disable_dynamic_updates()` 与 `disable_netbios()`）。
- 支持额外的 UDP 广播转发（`start_windows_udp_broadcast_relay()`）。
- 工程自带 Windows 第三方二进制如 `wintun.dll` 与 `WinDivert64.sys`。

#### Linux
- TUN 创建设备前会确保 `/dev/net/tun` 存在（`ensure_tun_device_node()`）。
- 支持 `socket_mark` 与网络命名空间（`NetNS`）。
- 公网 IPv6 提供器主要是 Linux 专属能力（[`public_ipv6_provider.rs`](easytier/src/instance/public_ipv6_provider.rs)）。
- 路由 / NDP 代理 / IPv6 forward 都有 Linux 条件编译分支。

#### macOS
- `tun` 在非 Network Extension 模式下会关闭 packet information。
- NIC 地址配置后需要补充本地路由（`cfg(all(target_os = "macos", not(feature = "macos-ne")))`）。
- fake TCP netfilter 有独立 macOS BPF 实现文件 [`macos_bpf.rs`](easytier/src/tunnel/fake_tcp/netfilter/macos_bpf.rs)。

#### Android
- 移动端不自己创建设备，而是由宿主 VPN Service 提供 TUN FD，再调用 `create_dev_for_mobile()` / `run_for_mobile()`。
- Android JNI 封装说明见 [`easytier-android-jni/README.md`](easytier-contrib/easytier-android-jni/README.md)。
- Android 不支持 ICMP 代理失败即中断。

以上 Windows/Linux/macOS 各平台相关逻辑均位于 [`easytier/src/instance/virtual_nic.rs`](easytier/src/instance/virtual_nic.rs)，除特别标注外。

---

## 4. 是否提供动态库输出？若有，导出 C 函数签名列表

### 4.1 结论

#### `easytier` 核心库本体
`easytier` 在 `[lib]` 中只声明普通 Rust 库，没有声明 `crate-type = ["cdylib"]`，因此**核心 crate 本体不直接提供面向 C 的动态库产物**。

#### 真正提供稳定 C ABI 的项目
- [`easytier-ffi`](easytier-contrib/easytier-ffi/Cargo.toml)：提供 `cdylib` + `rlib`，是 C ABI 的正式出口。
- [`easytier-android-jni`](easytier-contrib/easytier-android-jni/Cargo.toml)：提供 Android JNI `cdylib`，导出的是 `Java_*` JNI 符号，不是通用 C API。
- [`easytier-gui/src-tauri`](easytier-gui/src-tauri/Cargo.toml) 也构建 `cdylib`，但这是 GUI 宿主用途，不是核心网络库的 C API。

因此，如果问题是"EasyTier 核心库是否提供动态库形式输出，并列出导出的 C 函数签名"，答案应聚焦 [`easytier-ffi`](easytier-contrib/easytier-ffi/Cargo.toml)。

### 4.2 导出辅助类型
定义在 [`easytier-contrib/easytier-ffi/src/types.rs`](easytier-contrib/easytier-ffi/src/types.rs)：

```cpp
struct KeyValuePair {
    const char* key;
    const char* value;
};

using ConfigServerEventCallback = void (*)(const char* event_json, void* user_data);
```

### 4.3 `easytier-ffi` 导出 C 函数签名列表
以下函数均以 `extern "C"` 导出，定义在 [`easytier-contrib/easytier-ffi/src/lib.rs`](easytier-contrib/easytier-ffi/src/lib.rs)。签名使用现代 C++ 语法与 `<cstdint>` 固定宽度类型。

#### 网络实例管理 API

```cpp
int32_t parse_config(const char* cfg_str);

int32_t run_network_instance(const char* cfg_str);

int32_t retain_network_instance(const char* const* inst_names, uint64_t length);

int32_t delete_network_instance(const char* const* inst_names, uint64_t length);

int32_t list_instance(KeyValuePair* infos, uint64_t max_length);

int32_t collect_network_infos(KeyValuePair* infos, uint64_t max_length);

int32_t set_tun_fd(const char* inst_name, int32_t fd);

int32_t call_json_rpc(
    const char* service_name,
    const char* method_name,
    const char* domain_name,
    const char* payload_json,
    const char** out_response_json
);
```

#### 配置服务器客户端 API

```cpp
int32_t start_config_server_client(
    const char* config_server_url,
    const char* hostname,
    const char* machine_id,
    bool secure_mode,
    ConfigServerEventCallback callback,
    void* user_data
);

int32_t stop_config_server_client(void);

int32_t is_config_server_client_connected(void);
```

#### 数据面同步 API（由默认特性 `ffi-dataplane` 打开）

```cpp
uint64_t data_plane_tcp_connect(
    const char* inst_name,
    const char* dst_ip,
    uint16_t dst_port,
    uint64_t timeout_ms,
    const char** out_local_ip,
    uint16_t* out_local_port
);

uint64_t data_plane_tcp_bind(
    const char* inst_name,
    uint16_t local_port,
    uint64_t timeout_ms,
    const char** out_local_ip,
    uint16_t* out_local_port
);

uint64_t data_plane_tcp_accept(
    uint64_t handle,
    uint64_t timeout_ms,
    const char** out_local_ip,
    uint16_t* out_local_port,
    const char** out_peer_ip,
    uint16_t* out_peer_port
);

int32_t data_plane_tcp_read(
    uint64_t handle,
    uint8_t* buf,
    uint32_t len,
    uint64_t timeout_ms
);

int32_t data_plane_tcp_write(
    uint64_t handle,
    const uint8_t* buf,
    uint32_t len,
    uint64_t timeout_ms
);

int32_t data_plane_tcp_close(uint64_t handle);

int32_t data_plane_tcp_listener_close(uint64_t handle);

uint64_t data_plane_udp_bind(
    const char* inst_name,
    uint16_t local_port,
    uint64_t timeout_ms,
    const char** out_local_ip,
    uint16_t* out_local_port
);

int32_t data_plane_udp_send_to(
    uint64_t handle,
    const char* dst_ip,
    uint16_t dst_port,
    const uint8_t* buf,
    uint32_t len,
    uint64_t timeout_ms
);

int32_t data_plane_udp_recv_from(
    uint64_t handle,
    uint8_t* buf,
    uint32_t len,
    const char** out_ip,
    uint16_t* out_port,
    uint64_t timeout_ms
);

int32_t data_plane_udp_close(uint64_t handle);
```

#### 数据面异步操作控制 API

```cpp
int32_t data_plane_async_op_status(uint64_t handle);

int32_t data_plane_async_op_wait(uint64_t handle, uint64_t timeout_ms);

int32_t data_plane_async_op_cancel(uint64_t handle);

int32_t data_plane_async_op_free(uint64_t handle);

void data_plane_free_bytes(const uint8_t* ptr, uint32_t len);
```

#### 数据面异步开始/完成 API

```cpp
uint64_t data_plane_tcp_connect_start(
    const char* inst_name,
    const char* dst_ip,
    uint16_t dst_port,
    uint64_t timeout_ms
);

uint64_t data_plane_tcp_connect_finish(
    uint64_t op_handle,
    const char** out_local_ip,
    uint16_t* out_local_port
);

uint64_t data_plane_tcp_bind_start(
    const char* inst_name,
    uint16_t local_port,
    uint64_t timeout_ms
);

uint64_t data_plane_tcp_bind_finish(
    uint64_t op_handle,
    const char** out_local_ip,
    uint16_t* out_local_port
);

uint64_t data_plane_tcp_accept_start(uint64_t handle, uint64_t timeout_ms);

uint64_t data_plane_tcp_accept_finish(
    uint64_t op_handle,
    const char** out_local_ip,
    uint16_t* out_local_port,
    const char** out_peer_ip,
    uint16_t* out_peer_port
);

uint64_t data_plane_tcp_read_start(
    uint64_t handle,
    uint32_t max_len,
    uint64_t timeout_ms
);

int32_t data_plane_tcp_read_finish(
    uint64_t op_handle,
    const uint8_t** out_buf,
    uint32_t* out_len
);

uint64_t data_plane_tcp_write_start(
    uint64_t handle,
    const uint8_t* buf,
    uint32_t len,
    uint64_t timeout_ms
);

int32_t data_plane_tcp_write_finish(uint64_t op_handle);

uint64_t data_plane_udp_bind_start(
    const char* inst_name,
    uint16_t local_port,
    uint64_t timeout_ms
);

uint64_t data_plane_udp_bind_finish(
    uint64_t op_handle,
    const char** out_local_ip,
    uint16_t* out_local_port
);

uint64_t data_plane_udp_send_to_start(
    uint64_t handle,
    const char* dst_ip,
    uint16_t dst_port,
    const uint8_t* buf,
    uint32_t len,
    uint64_t timeout_ms
);

int32_t data_plane_udp_send_to_finish(uint64_t op_handle);

uint64_t data_plane_udp_recv_from_start(
    uint64_t handle,
    uint32_t max_len,
    uint64_t timeout_ms
);

int32_t data_plane_udp_recv_from_finish(
    uint64_t op_handle,
    const uint8_t** out_buf,
    uint32_t* out_len,
    const char** out_ip,
    uint16_t* out_port
);
```

#### 错误与内存管理 API

```cpp
void get_error_msg(const char** out);

void free_string(const char* s);
```

### 4.4 Android 动态库说明
Android 使用的是 [`easytier-android-jni`](easytier-contrib/easytier-android-jni/Cargo.toml) 输出的 `cdylib`。其导出符号是 JNI 形式，例如：

- `Java_com_easytier_jni_EasyTierJNI_setTunFd`
- `Java_com_easytier_jni_EasyTierJNI_parseConfig`
- `Java_com_easytier_jni_EasyTierJNI_runNetworkInstance`
- `Java_com_easytier_jni_EasyTierJNI_callJsonRpc`

这层更适合 Android App / VPN Service，而不是通用 C 调用。README 也明确说明其用途是 [`Android JNI 绑定库`](easytier-contrib/easytier-android-jni/README.md)。

---

## 5. 对官方文档准确性的补充判断

### 5.1 可直接信任的部分
- 根 README 的安装 / 基本用法示例大体可信，尤其是 [`README_CN.md`](README_CN.md) 中的快速开始与连接示例。
- `easytier/docs` 下的设计文档更适合理解未来或正在推进的实现方向，如 [`peer_conn_secure_mode_v3.md`](easytier/docs/peer_conn_secure_mode_v3.md)、[`relay_peer_manager_design.md`](easytier/docs/relay_peer_manager_design.md)、[`credential_peer.md`](easytier/docs/credential_peer.md)。

### 5.2 需要以源码为准的部分
- 实际 CLI 参数必须以 [`struct Cli`](easytier/src/core.rs:88) 与 [`struct NetworkOptions`](easytier/src/core.rs:144) 为准。
- 动态库能力不能只看 README，必须以 `crate-type = ["cdylib", "rlib"]` 和真实 `extern "C"` 函数为准。
- Android 集成文档中有少量 Java 路径文字与实际仓库结构不完全一致；源码实际 Kotlin 包位于 [`kotlin/com/easytier/jni`](easytier-contrib/easytier-android-jni/kotlin/com/easytier/jni/EasyTierJNI.kt)。

---

## 6. 调研结论摘要

- EasyTier 的"真正核心"在 [`easytier`](easytier/Cargo.toml)，它既是 Rust 库，也是 `easytier-core` 与 `easytier-cli` 的实现来源。
- 运行时是"多实例管理器 + 单实例 Peer/Tunnel/NIC/Route 运行时"结构，实例编排入口在 [`NetworkInstanceManager`](easytier/src/instance_manager.rs:30)，网络面核心在 [`PeerManager`](easytier/src/peers/peer_manager.rs:146)。
- 安全层围绕 Noise secure mode、peer session 复用、credential 身份和 relay 会话构建；设计文档主要集中在 [`easytier/docs`](easytier/docs/credential_peer.md)。
- 平台差异主要体现在 TUN 创建、系统服务、路由配置、防火墙和移动端 TUN FD 接入方式；Windows / Linux / macOS / Android 都有独立处理分支。
- 核心 crate 自身**不直接输出 C 动态库**；稳定的动态库封装来自 [`easytier-ffi`](easytier-contrib/easytier-ffi/Cargo.toml)。若需要跨语言嵌入，优先使用该库；Android 则使用 [`easytier-android-jni`](easytier-contrib/easytier-android-jni/Cargo.toml)。
