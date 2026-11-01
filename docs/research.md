# Labrix 最终调研与技术选型

研究日期：2026-11-01（Asia/Shanghai）。

状态：架构建议已形成；所有运行能力仍需 PoC 验证。

配套：[架构设计](architecture.md)、[实施路线](roadmap.md)、[来源及版本记录](sources.md)。

## 1. 最终判断

**值得创建 Labrix，但应把它定位为 GPU IDC 的多后端实验与验证工作台。第一版用现有开源引擎建立可复现的验证闭环，自研重点放在统一拓扑、设备能力描述、原生桌面操作、实验编排与结果解释。**

单一公开工具不能同时精确复现 B300 GPU、NVLink / NVSwitch、ConnectX-8、Q3400 XDR 数据平面、Mica 厂商 SONiC、Nexus 3524P 和 FortiGate 100F 的全部硬件与软件行为。这是本次查阅公开资料后的选型判断，不是对所有商业产品的穷尽性证明。

推荐组合：

| 层次 | 推荐选择 | 交付价值 |
| --- | --- | --- |
| 统一模型及快速检查 | 自研 Rust + Serde + petgraph | 设备/端口/线缆、平面隔离、能力匹配、规划版本 |
| 原生桌面 | GPUI Kit，先做 Windows PoC | 拓扑画布、属性面板、机柜视图、配置差异、实验时间线 |
| Ethernet 运行后端 | Containerlab + 必要时 vrnetlab / QEMU-KVM | SONiC、FRR、Linux 及授权厂商 VM 的互通实验 |
| 配置分析 | Batfish 外部服务；初期保留 pybatfish 桥接 | 静态可达性、路由及策略变化检查 |
| IB 结构及管理面 | Rust 图检查 + 可选 ibsim / OpenSM 沙箱 | rail、端口预算、拓扑发现及部分管理行为 |
| 性能建模 | 后续选择 ASTRA-sim / SimAI；必要时 ns-3 模型 | 工作负载、集合通信及网络瓶颈的模型比较 |
| 现实校准 | 小规模真实 GPU / HCA / 交换机测试环境 | RDMA、NCCL、PFC/ECN、SHARP、故障收敛的实测依据 |
| 资产输入 | NetBox 只读快照；CSV / Excel 导入 | 复用资产记录，保留来源和不确定性 |

**可以采用 GPUI Kit 实现 Labrix 自有软件的 Rust 架构。不能因此宣称整个实验环境是纯 Rust，也不能把 GPUI 的 GPU 渲染能力当成 GPU 硬件仿真能力。**

## 2. 输入资料、证据及适用范围

本次实际阅读了用户粘贴的既有调研，以及 Mica `network/网络模拟器/` 下的概述、GNS3、EVE-NG、BosonNetSim、H3C-HCL、VIRL、Packet Tracer、Containerlab 页面。Containerlab 页面当时为空，Boson / VIRL 页面内容很少，不能视作已完成的独立评估。内部资料仅用于确定问题与核查对象，公开文档不转载内部页面、截图、网络地址、凭据或完整生产配置。

本次附件是粘贴文本，**没有获得并重新解析原始 PDF**。其中提到的设备型号、32 台 GPU 主机、66 台存储节点及连接关系，视为用户提供的需求场景；不能据此确认实际数量、硬件版本、机柜位置或现网接线。本设计也不把 DGX 参考架构直接套用为 OEM HGX B300 的实际网卡配置。

文中使用四种证据状态：

- **官方事实**：项目文档、源码、许可证或 GitHub API 可直接支持。
- **需求输入**：用户资料提出，但本次没有现场核实。
- **评估判断**：根据能力、维护与成本作出的选择。
- **待实测**：必须由指定镜像、版本、环境及测试证明。

本次未启动网络实验、未安装 GPUI、未运行 GPU 性能测试、未连接生产设备。创建公开仓库和文档不代表平台已实现。

## 3. 对原调研的纠错与补充

| 原说法或隐含前提 | 本次结论 | 对 Labrix 的影响 |
| --- | --- | --- |
| Containerlab 只有 CLI / YAML、没有 GUI | 已过时。官方有 VS Code、Desktop 和 Web 入口 | Labrix 必须提供超出普通拓扑编辑器的价值 |
| DSX Air 不支持 NVLink | 不能作全产品断言。2026-03 官方文章提到 Spectrum-6 / NVLink switches；Config Manager 的特定 demo 则排除 IB / NVLink | 按产品、租户、镜像和 demo 范围核实，保留可选适配器 |
| ibsim 可用于 Q3400 的模拟 | 只能作为 IB 子网管理实验候选，不是 NVOS / XDR ASIC 仿真 | XDR、端口数量、属性与工具版本须独立做兼容性门禁 |
| SONiC-VS 支持某功能就等价 Mica 物理机 | 不成立。厂商分支、CLI、HWSKU、SAI、ASIC / SDK 都有差异 | 为具体镜像建立能力矩阵，不能只按 SONiC 品牌打勾 |
| Nexus 9000v 等同 Nexus 3524P | 不成立。仅能覆盖一部分 NX-OS 配置及协议行为 | 标记为替代平台，对 HSRP / LACP / vPC 等逐项验证 |
| FortiGate VM 能验证所有 100F 行为 | 软件功能取决于版本与许可，硬件加速、物理端口、吞吐不能替代 | 将策略、NAT、VPN、HA 拆成独立能力项 |
| EVE / 虚拟链路可证明生产无损网络性能 | 不成立。虚拟链路及队列模型不等于实际 buffer / ASIC | RoCE / IB 性能验收保留硬件测试 |
| Mininet 停在 2021 年，故项目已停止 | 不能由发行版本单独推断。API 显示仓库未归档且有 2024 年推送 | 评价维护风险，不使用“已死”等绝对表述 |
| 设置 800G 的链路即可测 800G | 只能表达规划速率或模型参数；虚拟机吞吐受宿主资源限制 | 分开记录 nominal / configured / measured 速率 |
| 纯 Rust 能消除异构运行环境 | 自有代码可统一语言，真实 NOS / 内核 / C/C++ 库仍存在 | 将语言统一边界写入架构决策 |

这些修正分别依据 [Containerlab GUI](https://containerlab.dev/manual/gui/)、[NVIDIA DSX Air 产品文章](https://developer.nvidia.com/blog/design-simulate-and-scale-ai-factory-infrastructure-with-nvidia-dsx-air/)、[Config Manager demo](https://docs.nvidia.com/switch-infrastructure/config-manager/getting-started/dsx-air-simulation-user-guide)、[ibsim README](https://github.com/linux-rdma/ibsim/blob/master/README)、[SONiC 虚拟节点](https://containerlab.dev/manual/kinds/sonic-vs/) 和 [Mininet 仓库](https://github.com/mininet/mininet)。

## 4. “真实模拟”需要拆成可验收的深度

| 验证类型 | 可以回答 | 不能自动推导 |
| --- | --- | --- |
| 结构模型 | 端口是否重复占用、线缆是否匹配、rail 是否接错、冗余路径是否共用故障域 | 实际线缆已这样连接 |
| 配置静态分析 | 模型支持范围内的路由、ACL、可达性及变更差异 | 协议真实时间、会话同步、ASIC 执行结果 |
| 真实 NOS 仿真 | 镜像实际接受的配置、协议状态与软件转发 | 同品牌另一平台/版本完全一致 |
| 离散事件 / 流量模型 | 指定假设下的作业完成时间、拥塞和拓扑比较 | 对某台 B300 的精确吞吐预测 |
| 硬件测试 | 指定设备、固件、负载与环境下的真实行为 | 任意规模、所有故障及未来版本均成立 |

这些类型是互补证据，不是一条“等级越高就覆盖所有低层问题”的直线。例如硬件 ping 成功不能证明所有 ACL 符合设计。报告应按具体断言展示证据，而不是只给一个笼统的“仿真精度 95%”。

## 5. 设备与能力覆盖

| 对象 | 推荐表达 / 运行方式 | 适合验证 | 关键边界与待验证项 |
| --- | --- | --- | --- |
| HGX B300 主机 | 物理规格模型 + Linux container / VM 替身 | IP、bonding、VLAN、MTU、服务及自动化流程 | GPU / NVSwitch / PCIe / 固件不由 Linux 替身复现；HCA 数量和 rail 来自实际 BOM |
| CPU / 管理服务器 | Linux container；需要内核隔离时 VM | DNS、DHCP、SSH、堡垒访问、路由与服务依赖 | 容器共享内核，不能等同独立操作系统 |
| 存储节点 | 网络端点 + 可选真实存储软件测试 | 存储网络隔离、会话、服务发现 | 虚拟盘吞吐不能证明物理存储 IOPS / 尾延迟 |
| Mica W6510 / W6920 / W6940 | 独立型号 profile + SONiC-VS / VM | 已验证镜像上的 config_db、FRR、VLAN、LAG 等 | 厂商 CLI、HWSKU、breakout、ACL 数据面、PFC / ECN 逐项验证；不模拟 Broadcom ASIC |
| Q3400 系列 | 物理结构模型；ibsim 候选管理后端 | 端口预算、rail 规划、部分 SM / SA / 诊断流程 | 无本次已证实可用的公开 Q3400 NVOS 虚拟镜像；不复现 SHARP / XDR 数据面 |
| CX8 HCA | 按板卡 / 端口 / PCIe 归属建模 | 规划、rail 与 NUMA / GPU 亲和关系检查 | veth 不是 HCA，不提供 mlx5、GPUDirect 或 IB verbs 能力 |
| FortiGate 100F | FortiGate-VM 作为功能替身 | 镜像许可允许的策略、NAT、路由、VPN、HA 实验 | 100F ASIC、吞吐、硬件端口和会话同步时序仍需真机 |
| Cisco Nexus 3524P | N9Kv 作为 NX-OS 功能替身 | 经验证的 VLAN、路由、LACP、HSRP 等配置 | 平台 / 版本功能集不等价；不复现 Algo Boost 及物理时延 |
| FortiWeb / DDoS / ALB | 厂商 VM 适配器或功能代理 | 特定 HTTP / 安全 / 流量路径断言 | Linux WAF / 代理只能证明通用路径，不能证明厂商规则语义一致 |
| BMC | Redfish 数据模型；可选 sushy-tools | 资产读取、开关机编排和引导流程 | 不复现厂商全部 OEM API、风扇、电源与硬件传感器 |
| 光模块 / 线缆 / PDU | 规格、连接及故障域模型 | 兼容性、端口拆分、供电依赖 | 不计算真实光预算、BER、温升与电力瞬态，除非后续有专用模型 |

依据：[SONiC container](https://containerlab.dev/manual/kinds/sonic-vs/)、[SONiC VM](https://containerlab.dev/manual/kinds/sonic-vm/)、[FortiGate 集成限制](https://containerlab.dev/manual/kinds/fortinet_fortigate/)、[N9Kv 集成](https://containerlab.dev/manual/kinds/vr-n9kv/)、[Virtual Redfish BMC](https://docs.openstack.org/sushy-tools/latest/user/dynamic-emulator.html)。表内“适合验证”是候选能力，不是本次实测支持清单。

Nexus 3500 的硬件特性和 N9Kv 的平台性质分别由 [Cisco 3500 数据表](https://www.cisco.com/c/en/us/products/collateral/switches/nexus-3548-switch/data_sheet_c78-707001.html) 与 [Nexus 9000v 概述](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/7-x/nx-osv/configuration/guide/b_Cisco_Nexus_9000v/b_Cisco_Nexus_9000v_chapter_011.html) 支持；后者是历史版本资料，具体可用功能仍应按目标 NX-OS 版本核对。

### 5.1 Q3400 的建模陷阱

官方 Q3400-RA 规格给出 **144 个 800Gb/s XDR 端口、72 个 OSFP 笼、最大吞吐 115.2Tb/s**。物理笼和逻辑端口不是一回事。必须把 chassis、内部 fabric device、cage、transceiver、logical port、lane / breakout 分开；具体内部交换设备关系由对应 SKU / NVOS 资料和实物确认。[NVIDIA Q32xx / Q34xx 手册](https://docs.nvidia.com/nvidia-q32xx-and-q34xx-xdr-800gb-s-infiniband-switch-systems-user-manual.pdf)

若假设 32 台服务器各有 8 个计算网端口，仅主机侧就是 256 个端点。这只是端口预算例子，不能从“256 / 144 向上取整”直接得出交换机数量：还要考虑上联、rail 隔离、双平面、收敛比和内部交换单元。该例子不确认实际 B300 板卡数量或接法。

### 5.2 Ethernet 与 IB 不能靠同名连线混为一种后端

IB 计算网采用独立端口类型、地址和管理语义。以太网 veth 不应连接到模型中的 IB 端口并显示“已成功仿真”。统一画布用于展示关系，Ethernet 运行与 ibsim 运行各自生成独立实验计划；第一版通过场景和证据组合，不做未经验证的跨引擎实时包交换。

可选 Soft-RoCE / RXE 能帮助验证一部分 RDMA 应用接口路径，但仍是 Ethernet 上的软件实现，不能等同 XDR InfiniBand、CX8 性能或 GPUDirect。rdma-core 明确列出 `rdma_rxe` 等 provider，接入仍需匹配宿主内核及权限。[rdma-core README](https://github.com/linux-rdma/rdma-core/blob/master/README.md)

## 6. 最值得借鉴的开源项目

以下是面向 Labrix 的主观优先级，依据需求匹配、接口可复用程度、维护状态和实施成本判断，不是性能排行。许可证快照及维护日期见 [核实记录](sources.md)。

### 6.1 P0：首轮重点研究或集成

| 项目 | 建议方式 | 最值得借鉴的部分 | 主要成本 / 限制 |
| --- | --- | --- | --- |
| [Containerlab](https://github.com/srl-labs/containerlab) | 首选运行后端，通过进程接口集成 | topology schema、kind 注册、端点映射、生命周期和 inspect | Linux 特权、镜像供应、版本兼容；它本身已提供 GUI |
| [vrnetlab](https://github.com/srl-labs/vrnetlab) | VM 型 NOS 的外部运行组件 | 镜像封装、串口启动、接口映射和就绪检测 | QEMU / KVM、较高内存、启动超时和厂商授权 |
| [netlab](https://github.com/ipspace/netlab) | 高优先级设计参考，必要时可选导出 / 调用 | 高层意图 → 地址 / 协议 / provider 配置；设备能力数据和模板 | Python / Ansible 生态；不要另起一套重复配置生成器 |
| [containerlab-app](https://github.com/srl-labs/containerlab-app) | 研究交互与文件兼容，保持互导 | 编辑 / 运行状态叠加、画布布局、console、draw.io 工作流 | TypeScript UI；不是直接装入 GPUI 的组件 |
| [Batfish](https://github.com/batfish/batfish) | 独立分析服务 | 配置快照、厂商中立模型、差异化可达性及反例 | Java 服务，常用 Python SDK；parser 支持不等于所有语义受支持 |
| [batfish/lab-validation](https://github.com/batfish/lab-validation) | 重点参考验证方法 | 用运行中的虚拟设备结果对照静态模型；记录配置与实际结果 | 每种镜像和功能需要自己的 fixture，不能复制结论 |
| [GPUI Kit](https://github.com/longbridge/gpui-kit) | Rust 原生桌面候选，PoC 通过后采用 | 成熟 UI 组件、面板、编辑和主题基础 | 拓扑画布、终端和大图交互仍需专门工程实现 |

netlab 的官方定位已经覆盖高层 YAML、多个 provider、配置生成和启动，因此 Labrix 的差异应集中在物理结构、IB / GPU 语义、证据和交互；不要把重写 netlab 作为第一里程碑。[netlab 文档](https://netlab.tools/)

Batfish 官方支持列表包含 Fortinet、SONiC、NX-OS、FRR；仍须检查 parse status、未识别配置和每个测试所依赖的模型能力。[支持列表](https://batfish.readthedocs.io/en/latest/supported_devices.html) [快照与查询](https://batfish.readthedocs.io/en/latest/notebooks/linked/getting-started-with-batfish.html)

### 6.2 P1：领域必需，但按阶段接入

| 项目 | 建议方式 | 借鉴价值 | 不应误用为 |
| --- | --- | --- | --- |
| [NetBox](https://github.com/netbox-community/netbox) | 只读导入和来源映射 | 设备、接口、线缆、机柜、模块的资产模型 | 仿真器、实时网络发现事实或待部署草稿库 |
| [SONiC buildimage](https://github.com/sonic-net/sonic-buildimage) / [sairedis](https://github.com/sonic-net/sonic-sairedis) | 官方 VS / VM 镜像及虚拟 SAI 实现参考 | 配置数据库、虚拟环境、硬件抽象的边界 | Mica 厂商分支或 ASIC 的精确复现 |
| [FRRouting](https://github.com/FRRouting/frr) | 自由软件实验 profile | BGP / OSPF 等真实协议进程，用作开源基线 | 任意厂商 CLI 或全部设备行为 |
| [ibsim](https://github.com/linux-rdma/ibsim) | 可选沙箱进程，先做兼容性验证 | MAD / SM / SA / PerfMgr、ibnetdiscover 格式、链路事件 | Q3400 NVOS、SHARP、RDMA 数据传输或线速性能 |
| [sushy-tools](https://docs.openstack.org/sushy-tools/latest/user/dynamic-emulator.html) | 后续 BMC 流程适配器 | fake / libvirt 驱动与 Redfish 控制 | GPU 服务器完整硬件或所有 OEM BMC 行为 |
| [BGPSim](https://github.com/nsg-ethz/bgpsim) | 可选 Rust 模型后端 | 快速路由规划、路由策略和事件队列 | 实际 NOS 兼容性、TCP / BFD / 协议报文精确时序 |
| [NeXosim](https://github.com/asynchronics/nexosim) | Rust 抽象仿真技术储备 | 离散事件、组件输入输出、事件注入与保存恢复 | 已完成的 Ethernet / IB / GPU 设备模型库 |

ibsim 官方描述是基于 MAD 的子网管理实验，使用 `libumad2sim` 拦截 libibumad 接口。本次 API 核实最新 release 为 `ibsim-0.12`（2022-10-31）；现代 rdma-core / OpenSM 及 XDR 属性必须单独验证，必要时只用 Rust 结构检查交付 IB 第一阶段。[ibsim README](https://github.com/linux-rdma/ibsim/blob/master/README) [Releases](https://github.com/linux-rdma/ibsim/releases)

BGPSim 官方明确简化协议且不建模 OSI 1–4 层；即使有可替换队列，也不能把其事件顺序结果当成真实收敛毫秒数。NeXosim 提供通用事件执行框架，具体网络模型要另写。[BGPSim README](https://github.com/nsg-ethz/bgpsim/blob/main/README.md) [NeXosim README](https://github.com/asynchronics/nexosim/blob/main/README.md)

### 6.3 P2：性能研究和替代实现参考

| 项目 | 价值 | 本次取舍 |
| --- | --- | --- |
| [ASTRA-sim](https://github.com/astra-sim/astra-sim) | AI 工作负载、集合通信、计算/内存/网络分层，支持 Chakra trace | 性能研究优先候选；以单一校准场景验证后再接入 |
| [SimAI](https://github.com/aliyun/SimAI) | AICB / SimCCL、分析和网络仿真组合 | 适合训练/推理通信规划；依赖链更长，不做首版必装组件 |
| [ns-3](https://www.nsnam.org/) | 分组级离散事件和可扩展协议模型 | 只有明确的拥塞研究问题才接入，RDMA 扩展及版本另审 |
| [SimGrid](https://simgrid.org/) | 集群、存储和分布式应用的系统级模型 | 适合大规模方案比较；不运行真实 NOS 配置 |
| [GNS3 server](https://github.com/GNS3/gns3-server) | GUI / controller / compute / emulator 分层和 API | 借鉴架构；现有 GNS3 用户可考虑后续兼容导入 |
| [Kathará](https://github.com/KatharaFramework/Kathara) | 轻量容器实验、L2 LAN 与实验描述 | 参考教学 / 小型实验体验，首版无需双运行后端 |
| [Mininet](https://github.com/mininet/mininet) | namespace / 虚拟链路 / SDN 原型 | 用于机制研究；不是多厂商 NOS 首选 |
| [egui](https://github.com/emilk/egui) / [Iced](https://github.com/iced-rs/iced) | Rust GUI、画布和不同渲染路径 | GPUI 不能满足 PoC 门槛时的备选，不同时维护三套 UI |
| [nccl-tests](https://github.com/NVIDIA/nccl-tests) / [rdma-core](https://github.com/linux-rdma/rdma-core) | 真实通信正确性及性能验证工具 | 用于后期硬件校准和结果导入，不是无 GPU 模拟器 |

ASTRA-sim 以工作负载和系统模型为入口；SimAI 组合了多个组件。两者均应作为独立批处理后端，输出带假设的预测，不直接接管真实 NOS 实验时钟。[ASTRA-sim README](https://github.com/astra-sim/astra-sim/blob/master/README.md) [SimAI README](https://github.com/aliyun/SimAI)

## 7. 为什么不以 EVE-NG 或 DSX Air 作为 Labrix 核心

**EVE-NG** 适合直接采购并使用的多厂商可视化实验平台。如果目的只是让工程师拖拽设备和打开 CLI，先评估它比从零做 Labrix 更经济。但 Pro 是商业产品，不能作为可任意复用的开源核心；它同样依赖厂商镜像，也没有据本次资料能证明的 Q3400 / CX8 硬件精确仿真。官方发布页核实到 `7.4.0-1`（2026-10-30），但该版本本身不改变硬件仿真边界。[EVE-NG Release Notes](https://www.eve-ng.net/index.php/documentation/release-notes/)

**DSX Air** 应保留为 NVIDIA 生态可选后端。2026-03 产品文章明确提到 NVLink switches，但这不证明它能复现当前 HGX B300 / Q3400 组合，也不证明每个租户可获得同样镜像。Config Manager demo 不包含 IB / NVLink 是该 demo 的边界。采用前需向 NVIDIA 确认目标设备、软件版本、API、数据范围、配额和授权；本次未登录租户实测。[产品文章](https://developer.nvidia.com/blog/design-simulate-and-scale-ai-factory-infrastructure-with-nvidia-dsx-air/) [demo 限制](https://docs.nvidia.com/switch-infrastructure/config-manager/getting-started/dsx-air-simulation-user-guide)

Packet Tracer、Boson NetSim、H3C HCL 和 Cisco CML 对教学、认证或对应厂商专项验证有价值。它们不作为本项目开源跨厂商核心；不沿用旧资料中的镜像下载站、旧节点上限或平台关闭指令作为实施依据。

## 8. GPUI Kit 与纯 Rust 架构评估

### 8.1 推荐结论

**条件采用 GPUI Kit，先进行一个有退出标准的 Windows 桌面 PoC。** 用户所说的 gpu-kit 已确认指 GPUI / GPUI Kit。Kit 是应用组件框架，GPUI 是底层 UI 基础；两者都不负责执行 GPU 计算模型或模拟 NVIDIA 硬件。

本次核实 GPUI Kit 最新 release 为 `v0.8.1`，2026-10-26 发布；其 crate manifest 声明 Apache-2.0。官方安装文档给出 Windows 10+、MSVC 工具链、Windows SDK、CMake 及当前依赖图所需 Rust 1.92+。此为上游要求，不代表 Labrix 已在这些平台测试通过。[v0.8.1 manifest](https://github.com/longbridge/gpui-kit/blob/v0.8.1/crates/kit/Cargo.toml) [安装文档](https://gpui-kit.com/docs/installation/)

Kit 提供的面板、表格、输入、编辑和主题基础适合密集的基础设施工具界面。**拓扑布局、端口级连线、连线避障、缩放、框选、撤销、终端状态和大图优化仍需 Labrix 实现或另选组件。** 不能把一套 UI 组件库当成现成网络拓扑编辑器。

### 8.2 “纯 Rust”的三个范围

| 范围 | 可行性 | 建议 |
| --- | --- | --- |
| Labrix 自有核心、CLI、桌面、API、Agent 均用 Rust | 可行 | 采用，尽量共享模型和协议类型 |
| 所有直接依赖和系统库均是 Rust，无 C/C++ / FFI | 不现实也非必要 | GUI、字体、TLS / SQLite 等依赖按实际构建审计，不作绝对承诺 |
| 所有仿真引擎、厂商 NOS 和 GPU 模型都重新用 Rust 实现 | 技术上可写局部抽象模型，完整目标成本极高 | 不纳入首版，不以语言纯度牺牲行为真实性 |

初版 Batfish 用 Java 服务 + 小型 pybatfish 外部桥接；vrnetlab 使用 Python / QEMU。应用进程间采用结构化 JSON，主程序不嵌 Python 解释器。若以后要移除 Python 桥接，先验证 Batfish API 版本与行为，再写 Rust 客户端。对外描述应是“Rust 原生应用，多引擎后端”。

### 8.3 备选 UI 的取舍

| 方案 | 评估优势 | 评估成本 | 决策 |
| --- | --- | --- | --- |
| GPUI Kit | 原生桌面、现成应用组件、Rust 统一 | 自定义画布和终端、依赖快照兼容、远程桌面渲染需测试 | 首选 PoC |
| egui / eframe | 原型速度、绘图自由、Rust / Web 复用 | 桌面应用级交互和复杂编辑体验需组装 | 首选备选 |
| Iced | 清楚的消息 / 状态模型、自定义组件及多渲染器 | 需自行组织复杂工作台和拓扑编辑 | 次选备选 |
| Rust + Web UI / Tauri | 浏览器图编辑生态和共享访问 | 不满足自有前端全 Rust 的偏好，多一套语言及打包边界 | 只有团队浏览器协作成为刚需时再评估 |

以上是工程判断；没有本次 GUI 性能测试支持谁“更快”。egui 和 Iced 的能力依据各自 [egui README](https://github.com/emilk/egui/blob/main/README.md)、[Iced README](https://github.com/iced-rs/iced/blob/master/README.md)。

### 8.4 GPUI PoC 必须回答的问题

1. Windows 本机及常用 RDP 环境能否可靠启动、显示中文、输入法组合输入、复制粘贴和 DPI 切换？
2. 1,000 节点 / 5,000 连线的合成拓扑，在目标机器能否达到交互预算？测试方法见路线文档。
3. 自定义画布和侧边表格是否能共享选择与增量状态，不阻塞后台实验事件？
4. 控制台在切换面板、隐藏窗口后是否保持后端会话？MVP 可先调用外部终端，嵌入终端单独评估。
5. 键盘导航、焦点、无障碍信息和错误展示是否可用？Kit 提供 AccessKit 支持，具体画布仍需应用接线和测试。[无障碍文档](https://gpui-kit.com/docs/accessibility/)
6. 锁定 Kit 版本后是否能稳定构建和打包？不同时混用不匹配的 GPUI upstream、gpui-pre 或其他 fork。

## 9. 技术选型决策

| 领域 | 首选 | 选择理由与限制 |
| --- | --- | --- |
| 语言与包结构 | Rust Cargo workspace | 同一模型供桌面、CLI、服务和 Agent 使用 |
| UI | GPUI Kit `=0.8.1` 作为首轮验证基线 | 用 Kit 对齐依赖；是否长期采用由 PoC 决定 |
| 异步与 API | Tokio + Axum；REST / JSON + WebSocket 事件 | 与外部工具进程隔离；首版无需 gRPC 和消息队列 |
| 类型与导出 | Serde / serde_json；JSON Schema | 稳定的 Labrix 模型，不把 clab YAML 作为全部领域真相 |
| 图算法 | petgraph；空间索引候选 rstar | 检查连通、冗余与故障域；不把普通图算法当网络协议模型 |
| 存储 | SQLite + SQLx；文件化 artifact | 单用户 / 单控制器起步，服务端并发需求出现后再迁移 PostgreSQL |
| Containerlab | 首轮锁定 `v0.82.0`，进程参数数组调用 | 采用 upstream engine，避免嵌入 Go 运行时 |
| 容器 / VM | Docker + QEMU/KVM via vrnetlab | Linux 运行端；宿主硬件虚拟化能力先检测 |
| 模板 | MiniJinja 候选；优先复用已审查模板 | 只生成有能力声明的配置，错误不能静默丢字段 |
| YAML | clab 适配层选一个仍维护的 parser 并验证 round-trip | 不默认选择已归档的 dtolnay/serde-yaml；首版内部交换用 JSON |
| XML / draw.io | quick-xml 候选；有损导入给出报告 | 图形元素不自动成为已确认物理连线 |
| NetBox / HTTP | reqwest + 明确的 snapshot importer | 首版只读，不由实验状态覆盖资产记录 |
| 密钥与日志 | 系统密钥存储 / 外部 secret ref；tracing | secret 不写入可提交拓扑、运行日志或证据包 |
| 性能模拟 | 外部批处理插件；首轮 ASTRA-sim PoC | 不在 GUI 线程运行，不与真实 NOS 时钟强行同步 |

除已核实的 Kit / Containerlab 基线外，其他 Rust 库是技术候选，**未安装、未锁版、未做兼容性或供应链审查**。实施时生成 Cargo.lock，核实 MSRV、许可证、构建平台与 advisories。serde-yaml 的归档状态见 [上游仓库](https://github.com/dtolnay/serde-yaml)。

## 10. 开源复用与发布边界

采用“自有实现 + 公开接口集成 + 独立分发依赖”的方式。Containerlab 为 BSD-3-Clause，vrnetlab 和 netlab 使用 MIT 文本，Batfish 和 NetBox 为 Apache-2.0；ibsim 为 GPL-2.0，GNS3 server 和 Kathará 为 GPL-3.0。GPL 工具放在独立进程并不自动豁免组合、修改、镜像打包或再分发义务；发布具体组合前按其文件和依赖审查。完整快照及上游链接见 [来源记录](sources.md)。

GPUI Kit 软件 manifest 为 Apache-2.0，部分上游文档另有 CC BY 4.0 说明；本报告仅独立总结事实与设计，不复制其文档段落、图片或组件实现。[软件许可](https://github.com/longbridge/gpui-kit/blob/main/LICENSE-APACHE) [文档许可说明](https://github.com/longbridge/gpui-kit/blob/main/LICENSE-DOCS.md)

厂商 VM 镜像和许可证由用户合法取得。不能因为 Containerlab / EVE / GNS3 支持某设备，就推断镜像允许公开分发或在任意平台运行；尤其不能把某平台订阅内的镜像权益自动扩展到别的平台。

公开仓库只放合成拓扑、模板、能力清单与脱敏研究。生产配置、IP / serial / GUID 清单、设备镜像、许可证、密钥和实际抓包保存在独立私有位置。

## 11. 首版的成功标准

首版用一个小型、多平面的合成 IDC 场景完成以下闭环：

**编辑 / 导入 → 模型检查 → 能力与资源预检 → 生成计划 → 启动 Ethernet 实验 → 收集证据 → 注入受控故障 → 比较预期 → 清理并重跑。**

主路径不需要 GPU，也不需要商业镜像：以 Linux / FRR 为可公开复现基线，SONiC 镜像单独验证；FortiGate / N9Kv 为用户自备镜像的扩展测试。IB 首阶段交付结构检查，ibsim 通过兼容性门禁后再开放管理实验。性能建模和真实硬件测试保留独立结果类型。

最有价值的差异是：工程师能清楚知道“哪条线接错了、哪个配置可能破坏路径、故障后哪些业务应受影响、结论基于什么证据、哪些部分仍需要真机测试”。若第一阶段只完成拖拽图标和打开 CLI，应优先复用 Containerlab GUI，而不是继续扩张自研范围。
