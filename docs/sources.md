# 调研来源与核实记录

核实日期：2026-11-01（Asia/Shanghai）。仓库推送 / commit 时间的原始值来自 GitHub API，以下日期列按 UTC 日期简记，不用于推断当天之后的变化。

## 1. 方法与证据边界

1. 阅读用户粘贴调研和 Mica `network/网络模拟器/` 目录的八个页面，提取需复核的判断。
2. 通过官方文档、上游 README / 源码 / 许可证、GitHub 仓库及 release API 复核选型相关事实。
3. 将公开事实、需求输入、工程判断和待实测能力分开。没有把第三方新闻、聚合站或搜索摘要作为最终技术结论依据。
4. 仓库状态使用 `GET /repos/{owner}/{repo}`；默认分支 commit 使用 commits API，版本使用 releases/latest。`pushed_at` 可能来自其他分支或标签，不等于默认分支最后提交，也不证明稳定性。
5. API 的 license 可能为 null 或 NOASSERTION；对 netlab、GPUI Kit、BGPSim 等重要候选补查了真实 LICENSE / Cargo manifest。其他复杂仓库保留“需逐组件审查”。

本次没有运行任何仿真器、厂商 NOS、GPUI 程序或硬件性能测试。表中的“未归档”“最近推送”只说明维护线索，不构成能力保证。

## 2. 关键来源与支持结论

### S01. Containerlab 核心与 GUI

- [核心仓库](https://github.com/srl-labs/containerlab)：运行引擎、许可和维护信息。
- [GUI 官方文档](https://containerlab.dev/manual/gui/)：VS Code / Desktop / Web 的共用拓扑工作流，含 draw.io / SVG 等功能入口。
- [containerlab-app 仓库](https://github.com/srl-labs/containerlab-app)：现有 UI 产品及 MIT 许可。
- [Link Impairments](https://containerlab.dev/manual/impairments/)：delay / jitter / loss 等实验损伤；不能据此推导 ASIC 队列或物理线速。
- [VM integration](https://containerlab.dev/manual/vrnetlab/) 和 [vrnetlab 仓库](https://github.com/srl-labs/vrnetlab)：QEMU VM 的容器封装及平台支持。

**核查中发现的文档差异**：VM integration 汇总表使用 `cisco_nexus9kv`，而锁定的 `v0.82.0` [节点注册源码](https://github.com/srl-labs/containerlab/blob/v0.82.0/nodes/vr_n9kv/vr-n9kv.go) 使用 `cisco_n9kv` / `vr-n9kv` / `vr-cisco_n9kv`。本设计首轮基线采用 `cisco_n9kv`。官方节点页面位于 [vr-n9kv](https://containerlab.dev/manual/kinds/vr-n9kv/)，不是按 kind 拼接出来的 URL。实现时再次用锁定 release schema 验证。

### S02. SONiC 与厂商替代边界

- [SONiC container](https://containerlab.dev/manual/kinds/sonic-vs/)：`sonic-vs`。
- [SONiC VM](https://containerlab.dev/manual/kinds/sonic-vm/)：`sonic-vm`、启动配置、接口映射和 guest / launcher 的执行位置区别。
- [sonic-buildimage](https://github.com/sonic-net/sonic-buildimage)：构建和镜像生态。
- [sairedis](https://github.com/sonic-net/sonic-sairedis)：SAI Redis / syncd 结构。
- [SONiC SWSS tests](https://github.com/sonic-net/sonic-swss/blob/master/tests/README.md)：虚拟交换测试无需真实设备或 ASIC，适合控制面行为验证。

**支持结论**：公开虚拟环境值得集成，但 Mica 厂商分支 / SKU / ASIC 必须建立自己的适配与验证清单。

### S03. FortiGate 与 Nexus

- [FortiGate kind](https://containerlab.dev/manual/kinds/fortinet_fortigate/)：集成测试版本为 7.0.14；文档提示 7.2+ 需有效许可和联网激活；VM 与端口映射。
- [N9Kv kind](https://containerlab.dev/manual/kinds/vr-n9kv/)：Nexus 9000v 集成与接口。
- [Cisco Nexus 3500 数据表](https://www.cisco.com/c/en/us/products/collateral/switches/nexus-3548-switch/data_sheet_c78-707001.html)：3500 平台 Algo Boost / ASIC 特性。
- [Nexus 9000v 官方概述](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/7-x/nx-osv/configuration/guide/b_Cisco_Nexus_9000v/b_Cisco_Nexus_9000v_chapter_011.html)：无特定硬件仿真；该页面为历史版本，只用于说明平台性质。
- [Nexus 9000v 10.5(x) Guide](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/n9000v-9300v-9500v/cisco-nexus-9000v-9300v-9500v-guide-release-105x.pdf)：平台与 hypervisor 限制，说明需按版本确认功能。

**未证明**：任意 FortiOS 7.4.x 镜像可免许可运行、VM 与 100F 完全一致、N9Kv 与 3524P 完全一致。实际授权条款与目标镜像未获取。

### S04. Q3400、IB 与真实通信

- [NVIDIA Q32xx / Q34xx 手册](https://docs.nvidia.com/nvidia-q32xx-and-q34xx-xdr-800gb-s-infiniband-switch-systems-user-manual.pdf)：Q3400-RA 的 144 × 800G 逻辑端口、72 OSFP 笼、115.2Tb/s 规格；具体 SKU 与内部结构另外核对。
- [ibsim README](https://github.com/linux-rdma/ibsim/blob/master/README)：MAD、SM / SA / PerfMgr、libumad2sim、发现格式与链路事件。
- [ibsim Releases](https://github.com/linux-rdma/ibsim/releases)：最新公开 release 为 0.12。
- [rdma-core README](https://github.com/linux-rdma/rdma-core/blob/master/README.md)：libibverbs / librdmacm / libibumad 与 provider，包括 rdma_rxe。
- [NCCL Tests README](https://github.com/NVIDIA/nccl-tests/blob/master/README.md)：NCCL 正确性和性能测试，依赖真实 CUDA / NCCL 环境。

**未证明**：ibsim 0.12 原生覆盖 XDR 扩展属性、Q3400 NVOS、SHARP 或真实 RDMA 数据面。没有找到并验证可公开运行的 Q3400 NOS 镜像；该结论仅限本次公开资料核查。

### S05. NVIDIA DSX Air 的范围差异

- [2026-03-16 官方产品文章](https://developer.nvidia.com/blog/design-simulate-and-scale-ai-factory-infrastructure-with-nvidia-dsx-air/)：Spectrum-6、NVLink switches、API / SDK 与云端实验工作流。
- [Config Manager DSX Air Simulation User Guide](https://docs.nvidia.com/switch-infrastructure/config-manager/getting-started/dsx-air-simulation-user-guide)：具体 demo 不含 IB / NVLink 工作流和部分 OS upgrade 路径。
- [Custom Topology](https://docs.nvidia.com/networking-ethernet-software/nvidia-air-v2/Custom-Topology/)：拓扑导入入口。

**支持结论**：不能将某 demo 的限制扩展成整个平台永久不支持 NVLink。也不能由产品文章推导具体 B300 / Q3400 / CX8 精确仿真或租户已具备访问能力。

### S06. 拓扑编译、资产与静态分析

- [netlab 官方文档](https://netlab.tools/)：高层拓扑、配置 / 地址 / provider 生成及生命周期。
- [netlab LICENSE](https://github.com/ipspace/netlab/blob/dev/LICENSE.md)：MIT 文本；API 未正确识别为具体 SPDX。
- [NetBox](https://github.com/netbox-community/netbox)：基础设施 source of truth 及 Apache-2.0。
- [Batfish 支持列表](https://batfish.readthedocs.io/en/latest/supported_devices.html)：含 NX-OS、Fortinet、SONiC、FRR。
- [Batfish Getting Started](https://batfish.readthedocs.io/en/latest/notebooks/linked/getting-started-with-batfish.html)：snapshot / vendor-neutral model / query 工作流。
- [batfish/lab-validation](https://github.com/batfish/lab-validation) 及其 [infra 文档](https://github.com/batfish/lab-validation/blob/main/infra/README.md)：真实虚拟设备实验与模型对照的方法参考。

**限制**：支持某厂商的解析不等于所有 NAT / ACL / HA / 路由语义均完整；需做 feature-level fixtures。

### S07. Rust 桌面与模型候选

- [GPUI Kit 仓库](https://github.com/longbridge/gpui-kit)、[v0.8.1 crate manifest](https://github.com/longbridge/gpui-kit/blob/v0.8.1/crates/kit/Cargo.toml)、[软件许可证](https://github.com/longbridge/gpui-kit/blob/main/LICENSE-APACHE)。
- [Kit 安装](https://gpui-kit.com/docs/installation/)：Windows / MSVC / SDK / CMake，Rust 基线及依赖对齐。
- [Kit 无障碍](https://gpui-kit.com/docs/accessibility/) 和 [打包](https://gpui-kit.com/docs/packaging/)：应用需要落实的具体工作。
- [Kit 文档许可](https://github.com/longbridge/gpui-kit/blob/main/LICENSE-DOCS.md)：部分文档的额外 CC BY 4.0 授权，不等于软件改成该许可。
- [BGPSim README](https://github.com/nsg-ethz/bgpsim/blob/main/README.md)、[crate manifest](https://github.com/nsg-ethz/bgpsim/blob/main/bgpsim/Cargo.toml)、[crate LICENSE](https://github.com/nsg-ethz/bgpsim/blob/main/bgpsim/LICENSE)：简化 BGP / OSPF 模型，Apache-2.0；不能建模真实 L1–4。
- [NeXosim README](https://github.com/asynchronics/nexosim/blob/main/README.md)：Rust 通用离散事件框架，MIT / Apache-2.0 双许可声明；不是现成网络设备仿真库。
- [egui README](https://github.com/emilk/egui/blob/main/README.md)：Rust immediate-mode GUI，MIT / Apache-2.0。
- [Iced README](https://github.com/iced-rs/iced/blob/master/README.md)：Rust 跨平台 GUI、自定义组件及渲染选择，MIT。
- [serde-yaml](https://github.com/dtolnay/serde-yaml)：已归档，不默认引入为新项目 YAML 层。

**未证明**：GPUI 在目标工作站 / RDP 上达到性能预算、已有可直接用的 IDC 画布、所有传递依赖均为 Rust、完整软件不需要任何 FFI 或系统库。

### S08. 性能及替代平台

- [ASTRA-sim](https://github.com/astra-sim/astra-sim)：分层 AI 系统模拟与 Chakra trace 输入。
- [SimAI](https://github.com/aliyun/SimAI)：AICB / SimCCL / ASTRA / ns-3 等组合及不同模拟路径。
- [ns-3 官方介绍](https://www2.nsnam.org/about/)：开源离散事件网络模拟；扩展模型独立核实。
- [SimGrid](https://simgrid.org/)：分布式应用和系统模型，LGPL。
- [GNS3 server README](https://github.com/GNS3/gns3-server/blob/master/README.md)：API、controller / emulator 边界。
- [Kathará README](https://github.com/KatharaFramework/Kathara/blob/main/README.md)：Docker 容器与虚拟 L2 LAN。
- [Mininet 仓库](https://github.com/mininet/mininet)：SDN 原型环境与维护线索。
- [Virtual Redfish BMC](https://docs.openstack.org/sushy-tools/latest/user/dynamic-emulator.html)：sushy-tools 的 fake / libvirt 等驱动。
- [EVE-NG Release Notes](https://www.eve-ng.net/index.php/documentation/release-notes/)：Pro 7.4.0-1 于 2026-10-30 发布。未据此沿用旧资料的节点上限或镜像分发权限。

**限制**：AI / 网络模拟器的结果取决于模型、版本与校准；本次没有获得 B300 性能 trace 或测量误差数据。

## 3. GitHub 仓库维护与许可证快照

以下仓库除特别标记外均 `archived=false`。链接指向上游仓库；精确记录可由其 [GitHub REST repository API](https://docs.github.com/en/rest/repos/repos#get-a-repository) 再次获取。许可证列是本次看到的顶层声明，不替代实际依赖树和文件级审查。

| 仓库 | 最近推送 UTC 日期 | 语言线索 | 本次许可证结论 |
| --- | --- | --- | --- |
| [srl-labs/containerlab](https://github.com/srl-labs/containerlab) | 2026-11-01 | Go | BSD-3-Clause |
| [srl-labs/containerlab-app](https://github.com/srl-labs/containerlab-app) | 2026-11-01 | TypeScript | MIT |
| [srl-labs/vrnetlab](https://github.com/srl-labs/vrnetlab) | 2026-10-18 | Python | MIT |
| [ipspace/netlab](https://github.com/ipspace/netlab) | 2026-11-01 | Python | LICENSE 为 MIT 文本；API 为 NOASSERTION |
| [GNS3/gns3-server](https://github.com/GNS3/gns3-server) | 2026-11-01 | Python | GPL-3.0 |
| [netbox-community/netbox](https://github.com/netbox-community/netbox) | 2026-10-31 | Python | Apache-2.0 |
| [batfish/batfish](https://github.com/batfish/batfish) | 2026-10-31 | Java | Apache-2.0 |
| [batfish/lab-validation](https://github.com/batfish/lab-validation) | 2026-10-28 | Python | Apache-2.0 |
| [linux-rdma/ibsim](https://github.com/linux-rdma/ibsim) | 2023-01-19 | C | GPL-2.0 |
| [sonic-net/sonic-buildimage](https://github.com/sonic-net/sonic-buildimage) | 2026-11-01 | C | API NOASSERTION，需逐组件审查 |
| [sonic-net/sonic-sairedis](https://github.com/sonic-net/sonic-sairedis) | 2026-11-01 | C++ | API NOASSERTION，需逐组件审查 |
| [FRRouting/frr](https://github.com/FRRouting/frr) | 2026-11-01 | C | API NOASSERTION，需逐文件 / 组件审查 |
| [mininet/mininet](https://github.com/mininet/mininet) | 2024-07-09 | Python | BSD-3-Clause |
| [KatharaFramework/Kathara](https://github.com/KatharaFramework/Kathara) | 2026-10-15 | Python | GPL-3.0 |
| [astra-sim/astra-sim](https://github.com/astra-sim/astra-sim) | 2026-10-29 | C++ | MIT，子模块另审 |
| [aliyun/SimAI](https://github.com/aliyun/SimAI) | 2026-09-20 | Python | Apache-2.0，组件另审 |
| [simgrid/simgrid](https://github.com/simgrid/simgrid) | 2026-11-01 | C++ | 官网 LGPL，API NOASSERTION，发布时核对具体文件 |
| [longbridge/gpui-kit](https://github.com/longbridge/gpui-kit) | 2026-11-01 | Rust | crate / LICENSE-APACHE 为 Apache-2.0；文档另有许可说明 |
| [zed-industries/zed](https://github.com/zed-industries/zed) | 2026-11-01 | Rust | API NOASSERTION；不得由整个 Zed 许可推断 GPUI crate 许可 |
| [emilk/egui](https://github.com/emilk/egui) | 2026-10-30 | Rust | README 声明 MIT / Apache-2.0 |
| [iced-rs/iced](https://github.com/iced-rs/iced) | 2026-10-31 | Rust | MIT |
| [NVIDIA/nccl-tests](https://github.com/NVIDIA/nccl-tests) | 2026-10-05 | CUDA | BSD-3-Clause |
| [linux-rdma/rdma-core](https://github.com/linux-rdma/rdma-core) | 2026-10-29 | C | API NOASSERTION，需逐组件审查 |
| [nsg-ethz/bgpsim](https://github.com/nsg-ethz/bgpsim) | 2026-06-13 | Rust | bgpsim/LICENSE 为 Apache-2.0，API license 为 null |
| [asynchronics/nexosim](https://github.com/asynchronics/nexosim) | 2026-10-22 | Rust | README 声明 MIT / Apache-2.0 |
| [dtolnay/serde-yaml](https://github.com/dtolnay/serde-yaml) | 2024-03-25 | Rust | Apache-2.0；**archived=true** |

没有以 star 数作为核心排名依据，也没有用一次最近推送宣称项目稳定或持续活跃。

## 4. 可追溯的版本与提交

| 对象 | 核实到的版本 / commit | 用途 |
| --- | --- | --- |
| Containerlab release | [v0.82.0](https://github.com/srl-labs/containerlab/releases/tag/v0.82.0)，2026-10-23 | 建议首轮 PoC 基线，非本次已安装 |
| Containerlab 默认分支快照 | [c5f567373912a718e167a1896fd9b47c88579e84](https://github.com/srl-labs/containerlab/commit/c5f567373912a718e167a1896fd9b47c88579e84) | 2026-11-01 核实记录，不等同 release |
| GPUI Kit release | [v0.8.1](https://github.com/longbridge/gpui-kit/releases/tag/v0.8.1)，2026-10-26 | 建议 UI PoC 基线 |
| GPUI Kit 默认分支快照 | [56e9153c4e80775cb6e29d5ef6f6b350bbab71b1](https://github.com/longbridge/gpui-kit/commit/56e9153c4e80775cb6e29d5ef6f6b350bbab71b1) | 与 release 分开记录 |
| ibsim release | [ibsim-0.12](https://github.com/linux-rdma/ibsim/releases/tag/ibsim-0.12)，2022-10-31 | 老版本兼容性风险依据 |
| ibsim 默认分支快照 | [db9c0e46bf23802e92cea5c72b5eea6d9075169a](https://github.com/linux-rdma/ibsim/commit/db9c0e46bf23802e92cea5c72b5eea6d9075169a)，commit 2022-10-31 | 与仓库 pushed_at 2023-01-19 不同 |
| Batfish lab-validation | [c206b9e6c1275a341f1f387bf946e571cbf60d79](https://github.com/batfish/lab-validation/commit/c206b9e6c1275a341f1f387bf946e571cbf60d79) | 实验对照方法参考 |
| netlab 默认 dev 分支 | [1b2b4876152c52a85451b6332fd35b0a204068b9](https://github.com/ipspace/netlab/commit/1b2b4876152c52a85451b6332fd35b0a204068b9) | 编译架构和模板参考；未选定发行版本 |

本次 netlab 与 lab-validation 的 releases/latest API 返回 404；这不说明它们没有 tags、包版本或可用代码。依赖真正进入实现时再固定可重现版本和 artifact digest。

## 5. 尚待实测的关键结论

- GPUI 大图、RDP、中文输入和控制台集成：只有验收计划，没有性能证明。
- SONiC / Mica 功能交集和 Batfish 语义覆盖：只有候选矩阵，没有支持认证。
- FortiGate HA / Nexus 替代场景：需要实际授权镜像与对照测试。
- ibsim 的现代 rdma-core / OpenSM / XDR 兼容：高风险 PoC，不把旧发布等同现代硬件支持。
- 多站点 / 大规模实验资源需求：只有预算方法，没有宿主容量检查。
- B300 / CX8 / Q3400 性能模型：没有校准数据，不能给出真实吞吐保证。

后续新增能力时，更新验证日期、输入、版本和实际结果；历史研究快照保留为当时的决策依据。
