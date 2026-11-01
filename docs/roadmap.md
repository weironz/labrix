# Labrix 实施路线与验收方案

日期：2026-11-01。以下工期为规划估算，不是承诺或已完成进度。按 1–2 名熟悉 Rust / Linux / 网络的工程师、可获得镜像及实验宿主估算；商业许可、GPUI 平台问题和模型兼容性可能使工期扩大。

## 1. 分阶段范围

| 阶段 | 建议预算 | 交付范围 | 完成条件 |
| --- | --- | --- | --- |
| A：高风险 PoC | 1–2 周 | GPUI 画布；clab 生命周期；Batfish 解析；ibsim 兼容性 | 对每条候选路线给出 go / conditional / no-go 和可复现证据 |
| B：Rust 模型与 CLI | 2–3 周 | 类型、导入、端口 / rail / 冗余检查、plan 和 diff | 合成多平面拓扑可确定性生成并发现已知错误 |
| C：运行与证据闭环 | 3–4 周 | 单 Linux Agent、clab adapter、运行状态、采集、故障与清理 | 一次实验可复现、恢复中断、产生可信断言结果 |
| D：桌面 MVP | 3–4 周 | 拓扑 / 属性 / 问题定位、配置差异、运行面板 | 可完成导入至报告全过程，外部 console 可用 |
| E：IB / 厂商扩展 | 2–4 周 | 通过门禁的 ibsim；自备镜像的 FortiGate / N9Kv；BMC 可选 | 功能逐项有 fixture、版本及限制；不支持项可见 |
| F：性能及硬件校准 | 单独立项 | ASTRA-sim 或 SimAI 首选其一，RDMA / NCCL 对照 | 参数、留出场景和误差报告齐全 |

A–D 顺序预算约 9–13 周，允许部分工作重叠但不以“并行”掩盖依赖。首个可交互原型应远早于完整 MVP。E / F 不应成为 A–D 完成的必要条件。

## 2. 阶段 A：四项先验技术验证

### A1. GPUI Kit / Windows

基线从 `gpui-kit = "=0.8.1"` 开始，提交 Cargo.lock；使用与其匹配的 GPUI 依赖。此为验证建议，不是当前仓库已经存在的依赖配置。[上游安装要求](https://gpui-kit.com/docs/installation/)

验证内容：

- 原生窗口、中文字体和中文输入法、剪贴板、拖拽、滚动、150% / 200% DPI。
- 本机与用户常用 RDP 场景分别执行，记录 GPU / driver / Windows / 分辨率。
- 合成 100、1,000、5,000 节点层级数据；典型压力点取 1,000 节点 / 5,000 连线，含端口详情和状态更新。
- 连续缩放、框选、拖动、搜索定位、100 条/秒状态事件，并发运行后台任务。
- 记录 warm-up 后 60 秒的 frame time p50 / p95 / p99、输入响应、CPU / 内存和观察到的卡顿。

**建议验收目标**：目标工作站上典型交互 p95 frame time ≤ 33ms，选择 / 搜索反馈 ≤ 100ms；10 分钟状态更新无持续内存增长或丢失选择。它们是产品预算，尚无实测结果。达不到时先验证视口裁剪、文本缓存和批量绘制；不能在 2 周 PoC 内收敛则切换 egui 备选并重新测量。

### A2. Containerlab + SONiC / VM

用锁定 `v0.82.0` 的 Linux 宿主验证 Linux / FRR 基线，再评估一个具体 SONiC VS 或 VM 镜像。成功条件不只是“容器 running”，必须读取 guest 版本、接口和协议状态。

记录：镜像 digest、宿主 kernel / Docker / clab 版本、KVM 检测、启动就绪时间、节点 RSS、CPU、接口映射、清理残留。VM 场景证明采集在 guest 内执行，不能误取 launcher shell。[SONiC VM 执行位置](https://containerlab.dev/manual/kinds/sonic-vm/)

许可证未具备时 FortiGate / N9Kv 记为 blocked-by-image-or-license，不阻断自由软件基线。对 Containerlab 文档中的版本和 kind 不一致，以锁定 release 的 schema / 注册源码为准；当前 v0.82.0 的 Nexus kind 为 `cisco_n9kv`。[注册源码](https://github.com/srl-labs/containerlab/blob/v0.82.0/nodes/vr_n9kv/vr-n9kv.go)

### A3. Batfish 支持是否覆盖实际断言

取脱敏且有已知预期的 FRR、SONiC、NX-OS 和 FortiOS 样例。记录 parse status、警告、未识别行和关键字段是否进入模型；对至少一个路由和一个 ACL/NAT 断言与运行中的实验结果对照。

通过条件：相关语义可解释、关键配置未被忽略、有一个故意错误能被检测。未完全解析时按断言相关性降为 unknown，不能只因为支持列表有厂商名就通过。[Batfish 支持表](https://batfish.readthedocs.io/en/latest/supported_devices.html) [运行结果对照参考](https://github.com/batfish/lab-validation)

### A4. ibsim / XDR 兼容性

先构建最小 CA–switch–CA fabric，配合固定版本 OpenSM 和 rdma-core 验证拓扑发现、SM 启动、端口 up/down 及恢复；再测试大端口数量和目标导出字段。

必须回答：

1. 当前工具链是否可编译和运行 `ibsim-0.12`？
2. 输入能否表示需要的拓扑规模、GUID、端口号和交换组件？
3. XDR 速率 / 扩展属性是否接受，是否被降级或忽略？
4. 输出与预期一致，还是仅能输出旧制式属性？

不能通过时：IB 第一阶段保持 Rust 拓扑检查；允许以“通用 IB 管理模型”运行受限实验，但禁止命名为 Q3400 精确模拟。不要为维持功能清单而默认 fork 并重写全部新标准支持。[ibsim 官方范围](https://github.com/linux-rdma/ibsim/blob/master/README)

## 3. 最小实验场景

公开样例使用合成命名和文档保留地址，不包含真实 IDC 标识。

### 场景 1：自由软件 Ethernet 基线

Linux 业务端点、管理端点、两台 FRR 路由节点和一个隔离业务端点，共约 6–8 个轻量节点。验证：接口、VLAN（按具体 Linux / switch profile）、路由、管理隔离、链路中断后的剩余路径及恢复。

先证明模型、计划、运行、采集与清理流程，再加入两台指定 SONiC 镜像验证交换配置。FRR profile 不伪装成 Mica CLI。

### 场景 2：厂商边界扩展

两台 FortiGate-VM、两台 N9Kv、内部 Linux 客户端、外部服务和隔离管理端点。分别验证 NAT、策略、VLAN、LACP、网关协议及 HA，按镜像功能和许可决定测试子集。

HA 不能只测单次 ping；至少包含已有 TCP 会话、新建会话、回程路由、故障检测和恢复。N9Kv 通过不等于 Nexus 3524P 已验证；实验报告写明替代平台。

### 场景 3：IB 结构 / 管理

2–4 个抽象服务器、每台多个 HCA 端口、两个以上 rail 和独立的 BMC 平面。故意制造 rail 交叉、同端口重复接线、cage / logical port 混淆和单故障域依赖，结构检查必须能定位错误。

ibsim 通过 A4 后再增加 SM / discovery / link event 实验，保存其真实支持的字段范围。不得把该场景的管理流量误称为 RDMA 业务流量。

## 4. 验收矩阵

| 检查 | 方法 | 通过证据 | 不通过 / 未知时表现 |
| --- | --- | --- | --- |
| 模型身份与导入 | 同设备不同来源、同名不同 serial 的合成 fixture | 明确冲突，不误合并 | review item，保留两条来源 |
| 端口 / breakout | 互斥组合及重复占用 | 错误定位到端口与线缆 | 阻断生成执行计划 |
| 平面类型 | 尝试 Ethernet 直连 IB | 类型诊断 | 阻断；不自动转成 veth |
| 配置和接口映射 | 主机 / SONiC / VM 逐项读取 | manifest 与实际接口一致 | 启动未就绪或采集错误可见 |
| 可达性 / 隔离 | 正反断言、抓包或路由信息 | 预期与观测一致 | fail 与 error 分开 |
| Batfish 覆盖 | 含故意未识别语句的快照 | parser 警告与相关性检查 | unknown，不能绿色通过 |
| 链路故障 | 注入、检查、撤回、再次检查 | 前后证据与事件时间齐全 | 恢复失败使 run 失败 |
| 客户端断线 | GUI 关闭 / WS 重连 | 运行继续，cursor 补事件 | stale 可见，不显示伪 down |
| Agent / 进程中断 | 中断 provision / collect / cleanup | reconcile 找到实际状态 | 不重复创建，不误删其他资源 |
| 运行取消 | 用户取消启动中实验 | 子进程停止与清理证据 | 残留资源清单可重试 |
| 地址冲突 | 宿主路由与实验管理网重叠 | 预检拒绝或选择新子网 | 不启动冲突网络 |
| 可复现 | 同 revision 重跑两次 | 配置 / plan hash 一致，结果可解释 | 随机种子、外部变化被记录 |
| 公开导出 | 合成 secret / 实体标识测试 | 脱敏生效、原始文件私有保存 | 阻断公开报告生成 |

## 5. 规模与资源预算

不能把模型节点数和同时运行 VM 数当成一个指标。三种规模分别评估：

- **编辑规模**：数千设备和更多端口 / 连线，主要受画布与模型算法影响。
- **轻量运行规模**：Linux / FRR 等容器，主要受宿主 CPU、内存、接口数量和实验流量影响。
- **重型运行规模**：SONiC VM / N9Kv / 防火墙 VM，资源占用和启动时间取决于镜像。

初始实验宿主可按 16–32 CPU 核心、128GiB RAM、NVMe 和可用 `/dev/kvm` 做采购前预算；这是小型混合 VM 实验的估算建议，不证明当前任何宿主具备资源，也不是 32 台 GPU + 66 台存储全量运行的容量保证。

按实际 profile 测量后估算：

```text
required_memory = Σ(instance_count × measured_node_memory)
                + controller/agent/tools overhead
                + temporary boot peak
                + configured safety reserve
```

CPU 同时考虑 VM vCPU 需求、启动峰值与流量负载，不能只做静态核数相加。磁盘考虑原始镜像、写入层、运行快照、pcap 与 evidence retention。资源不足时缩小运行切片或串行执行场景，完整物理拓扑仍可保留在模型中。

32 / 66 规模先做“全量结构校验 + 代表性运行切片”，有观测数据后再决定多宿主。首版不因节点多就直接引入 Kubernetes / Clabernetes。

## 6. 性能研究的进入条件

只有出现明确决策问题才进入阶段 F，例如：改变 rail 布局、上联收敛比、故障后带宽或集合通信算法，对指定工作负载完成时间有什么影响？

先选 ASTRA-sim 或 SimAI 之一，避免同时维护多个相似工具链。要求：

1. 一个最小模型可以复现上游样例。
2. Labrix 输入转换不会遗漏关键假设。
3. 模拟器版本、workload、拓扑、随机种子和 profile 可固定。
4. 真实小集群测试用于校准，独立留出场景用于验证。
5. 报告给出误差分布、适用区间和误差来源。

若无法取得 GPU 硬件，仍可交付相对拓扑比较，但结果必须标为未校准的模型预测。RDMA / NCCL 性能测试是后期独立授权的测试工作，本次仅设计方法，没有执行。

## 7. 开发与发布检查

实施时使用常规 Rust 格式、编译和 lint 检查，并为确有行为风险的部分建立测试：模型约束、状态机 / 恢复、能力降级、端点映射、资源归属和证据判定。GUI 低风险样式调整不需要用镜像式单元测试固化实现细节。

CI 分层：纯模型与编译测试在普通 runner；Linux / KVM / NOS 集成测试在专用隔离 runner；商业镜像测试手动触发并从私有存储取得镜像。公开 CI 不上传原始生产配置、secret 或设备镜像。本仓库当前没有配置自动 CI，以免文档初始提交触发无关构建。

每个 capability 的 fixture 至少包含：支持版本、最小拓扑、配置输入、预期断言、实际输出、已知限制和最后验证日期。升级镜像、clab 或解析器后，只重跑受影响的能力集合。

## 8. 优先解决的开放问题

| 问题 | 当前建议 | 决策证据 |
| --- | --- | --- |
| Mica 厂商 SONiC 能否提供 VM / VS 或测试镜像 | 官方 VS 先验证，厂商差异单独 profile | 镜像及版本、厂商功能清单、PoC 输出 |
| 目标 Q3400 / HCA 的准确 SKU / 内部组成 | 先保留 unknown，不从 PDF 推断 | BOM、手册和实际 inventory / port mapping |
| FortiOS 目标版本与 VM 许可 | 用户自备授权镜像，独立测试 | 激活结果、许可范围和 HA 功能实测 |
| N9Kv 与 3524P 的实际交集 | 建立替代映射和逐项断言 | NX-OS 版本 / 功能清单及真机对照 |
| GPUI 目标平台兼容性 | Windows 先行，跨平台不先作发布承诺 | A1 的输入法、RDP、大图与打包记录 |
| 是否需要浏览器协作 | 桌面 + API 起步 | 明确的多人工作流与访问需求 |
| 性能模型能否取得 B300 profile | 先相对分析，待硬件校准 | workload trace、GPU/HCA/固件信息及留出验证 |

这些问题不阻止先实现纯模型与自由软件实验闭环；只有对应扩展功能依赖其答案。
