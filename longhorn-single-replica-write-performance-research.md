# Longhorn 本地单副本写入性能研究方案

状态：研究立项与实验设计；尚未执行性能实验或实现性能补丁。源码核对日期：2026-11-11。

## 1. 研究问题与边界

研究问题：在同一套 RAID 0／高速本地存储上，Longhorn 本地单副本的写入性能损失分别发生在哪里？能否通过缩短数据路径、降低每次 I/O 的 CPU 成本与提高并发效率，在保留约定的数据语义下接近底层设备？

第一阶段固定为一个卷、一个副本、应用与副本同节点，随后验证多卷竞争和 Harvester 虚拟机。远程单副本作为对照，受网络带宽、往返延迟约束，单独报告。单副本不自动意味着本地副本。

主研究不叠加跨节点条带化、纠删码或多副本复制。这些问题改变数据布局及故障模型，适合后续独立课题。RAID 0 没有校验写惩罚；底层越快，固定的软件开销越容易成为瓶颈。

研究产出包括可复现实验、瓶颈证据、至少一个有测量依据的原型，以及正确性验证。结果允许是“某项优化无显著收益”，不能预先承诺任何负载都达到裸盘性能。

## 2. 基线版本与已核对事实

### 2.1 本项目的版本关系

- 当前 Harvester 分支为 `wnt/v1.8`。
- `scripts/wnt/wnt-common` 的 Longhorn manager 打包基础版本为 `v1.11.2`，存在自定义镜像版本后缀。
- 当前 `go.mod` 引用 manager `v1.10.0`、engine `v1.10.0-rc3`。这些编译依赖不能证明现场运行的 engine 版本。
- 下述 V1 源码分析使用上游 `longhorn-engine/v1.11.2`；本地 vendor 与上游 manager `v1.11.2` 均核对了 strict-local 的 UNIX socket 参数。

正式实验必须记录实际 manager、engine、instance-manager、CSI 与 SPDK 组件的镜像 digest、源代码 commit、内核、固件、磁盘型号、RAID 参数、文件系统及挂载参数。无法映射到确定源码的镜像不作为补丁归因的正式基线。

### 2.2 V1 写路径与可检验假设

常见 V1 blockdev 路径可概括为：

```text
应用／VM → Linux 块层／iSCSI／TGTD → engine frontend
        → controller／replicator → dataconn TCP 或 UNIX socket
        → replica → 活动稀疏文件／宿主文件系统 → RAID 0 → 磁盘
```

实际入口、进程及协议以实验拓扑和跟踪结果为准。VM 额外经过 guest 文件系统、virtio、QEMU 等环节，应分层测量。

| 已核对代码 | 事实 | 待检验的性能假设 |
| --- | --- | --- |
| `controller/multi_writer_at.go:MultiWriterAt.WriteAt` | 单 writer 仍使用通用的切片分配、goroutine 和 WaitGroup 路径 | 单副本直接调用能否显著减少小块 I/O CPU 成本 |
| `dataconn/client.go` | 请求经过 channel、序列号映射、完成通知；有集中处理循环 | 高 IOPS 下是否存在单个循环饱和、排队或分配瓶颈 |
| `backend/remote/remote.go` | 支持 TCP 和 UNIX socket；连接数常量为 2 | 协议处理与进程边界成本占比多少，多队列是否有效 |
| `replica/diff_disk.go` | 对齐写进入活动文件；非对齐端点有读改写及互斥锁 | 实际业务是否存在导致额外读写／锁竞争的非对齐请求 |
| `replica/revision_counter.go`、`replica/replica.go` | 启用 revision counter 时存在额外更新与等待路径 | 实际卷是否启用，开销占比及关闭后的恢复语义影响 |
| manager `engineapi/instance_manager.go` | strict-local 为 engine、replica 设置 `--data-server-protocol unix` | 现有本地优化后，剩余通信成本是否仍足以支持进一步合并路径 |

不能从 `RLock` 推断整个写路径串行，也不能把所有写都描述成快照链读改写。上述代码支持并发；是否受限需要每线程 CPU、队列和时延证据。

revision counter 的默认设置、StorageClass 参数和实际 Volume spec 应分别核对；不能假定它在现场开启，也不能把关闭它作为默认优化。

### 2.3 V2 与 UBLK 的已有工作

Longhorn 官方性能调查已讨论 TGTD／iSCSI、数据复制与分配，以及 V2 内部 NVMe-oF 层次带来的开销；UBLK 是必须纳入的现有对照，不能当作本课题的新发明。调查中的数值只说明其测试环境，不能作为本设备的预期结果。

以 1.11.2 文档为参照，V2 为 Technical Preview；UBLK 自 1.9 引入，要求内核至少 6.0 且具备 `ublk_drv`。使用前需核对目标 Harvester OS、功能支持及已知故障，特别是 instance-manager 异常后的设备清理问题。

V2 的原生 PCI NVMe bdev 和经 `/dev/...` 路径使用的 AIO bdev 不能混为一谈。硬件 RAID 逻辑盘或 Linux md RAID 不能直接假设具有单个原生 PCI NVMe 的 SPDK 路径。研究首先固定同一 RAID 设备；原生 NVMe 另列硬件实验。

尚未完成目标 SPDK engine 对应版本的本地 shortcut 源码核对。因此“合并 V2 本地路径”只是候选方向，实现前必须查明已有 bdev 拓扑，避免重复上游工作。

## 3. 怎样定义“接近裸盘”

以相同硬件、工作集、负载、持久化要求和固定 CPU 预算为条件，分别报告吞吐、延迟和资源成本。单个顺序带宽数字不能代表全部写能力。

下表为暂定研究目标，待基线测量后确认是否可行；它们不是已实现的结果或产品承诺。

| 指标 | 暂定目标 | 约束 |
| --- | --- | --- |
| 大块顺序写带宽 | 达到同 RAID 设备 R0 的 90% | 稳态、同持久化语义、固定 CPU 预算 |
| 4 KiB 随机写 IOPS | 在选定 QD 下达到 R0 的 80% | 报告实际 QD、numjobs、P99，不靠无限增大并发 |
| QD1 写延迟 | 普通写及持久化事务分别争取不超过 R0 的 1.25 倍 | 单独报告绝对微秒差；若达不到，解释固定路径成本 |
| CPU 成本 | 满足预先约定的整节点核数预算 | 报告 CPU-seconds/GiB、cycles/I/O 和空闲轮询成本 |
| 基线回归 | 选定核心负载无超过 5% 的稳定回退 | 结合重复实验离散程度，不能只比较最佳值 |

主要接受对象为本地单副本指定负载。快照、备份等能力是否同时达标必须明确写入结果，不能用功能缩减的原型代替完整 Longhorn 性能结论。

## 4. 实验分层与对照

### 4.1 测试层次

| 层次 | 测量对象 | 回答的问题 |
| --- | --- | --- |
| R0 | 专用可擦除测试 RAID 块设备，direct I/O | 硬件在相同负载下的参考上限 |
| R1 | 同设备上的宿主文件系统测试文件 | 文件系统、分配及文件访问代价 |
| R2 | V1 replica 存储层微基准，使用受控测试副本 | 稀疏文件／索引／快照逻辑代价 |
| R3 | V1 socket frontend／协议入口及完整 blockdev | engine 通信与块设备前端代价 |
| R4 | 宿主／Pod 访问完整 Longhorn 设备 | 完整主机端表现 |
| R5 | Harvester VM 内的相同负载 | guest、virtio、QEMU 与存储组合表现 |

R2、R3 需要测试工具适配或埋点，必须记录其入口绕过了哪些语义。各层差值不能直接相加归因，因为并发、缓存和批处理会相互影响。

对每个相同负载，同时给出 `candidate/R0`、`candidate/R1` 和 `candidate/original-Longhorn`。V2 改用块设备时，文件系统开销消除属于架构变化，不能伪称为纯通信优化。

### 4.2 首轮最小矩阵

1. 同一节点、同一设备，比较 R0、R1、V1 普通单副本本地放置、V1 strict-local。
2. 条件满足时比较 V2 默认前端与 UBLK，记录实际 bdev 类型及路径。
3. 选择最主要瓶颈进行单变量原型 A/B；确认收益后再组合优化。
4. 保留远程单副本样本，确认本地路径变化的适用范围。
5. 最后进入 VM 和多个卷竞争的场景，检查单卷提速是否以牺牲其他卷为代价。

strict-local 改变调度与使用约束，不能直接当作普通 locality 模式的透明替换；特别需核对迁移、attach 与升级能力。

### 4.3 工作负载与实验控制

- 4 KiB 随机写：QD 1、4、16、32、64；先固定 `numjobs=1`，随后独立研究多 job。报告 `iodepth × numjobs` 及实际达到的深度。
- 128 KiB／1 MiB 顺序写：区分 QD1 延迟与足够并发下的饱和带宽。
- 普通 direct 写、每次写后 fsync／fdatasync、分组同步分别测试。同步 ioengine 的名义高 iodepth 不代表真实高并发；同步事务统计从写开始到同步完成的完整时延。
- 分开测首次稀疏分配、预填充覆盖写，以及固定快照深度，例如 0、1、8。不要预设快照数一定影响对齐覆盖写。
- 工作集覆盖实际映射／缓存行为；预填充和 SSD 预处理足以暴露 SLC 缓存耗尽后的稳态。记录设备写入历史和剩余空间。
- 固定 I/O 对齐、随机种子、数据可压缩性、CPU／NUMA、频率策略、RAID stripe 和 cache policy。随机负载避免全部零数据。
- 推荐先以 60 秒预热、300 秒计量起步；若设备尚未稳态则延长并记录判据。每组至少 5 次有效重复，A/B 顺序交替并随机化，报告中位数、范围及样本数。
- 监视温度、降频、后台备份／快照合并／重建和 GC。异常样本记录原因，不能只删除慢结果。不要通过全局清缓存干扰共享工作负载。
- 所有写测试仅在明确分配的可擦除测试设备、卷和 VM 中进行。本方案未对当前工作站或现场磁盘执行写测试。

## 5. 持久化与指标口径

`O_DIRECT` 主要约束页缓存路径，不等于写入已抵抗掉电。应用普通写完成和 flush／FUA／fsync 完成的保证不同。

R0 与 Longhorn 必须一致记录设备易失写缓存、控制器电池／电容保护、flush 支持及完成语义。任何通过提前 ACK、忽略 flush 或改变 cache policy 获得的结果必须单独标记，不能进入等语义性能结论。

每个运行至少保存：

- fio 版本、完整 job、JSON 输出、实际 I/O 深度，带宽、IOPS、P50／P99／P99.9。区分提交、完成、总时延和同步操作时延。
- 逐线程 CPU／cycles、上下文切换、迁核、队列等待、分配与 GC；Go 路径使用可控开销的 pprof，系统路径使用 perf／eBPF。
- 设备服务时延、队列、逻辑及底层物理写入字节、网络流量。RAID 的写放大统计口径须注明观测层，避免重复计算逻辑盘和成员盘。
- V2 轮询线程的空闲占核、hugepages 与 NUMA 位置；不能只以 Go 进程 CPU 判断全栈成本。
- 带／不带采样器的成对测试，确认跟踪开销。高速 NVMe／RAID 的 `iostat %util=100` 本身不足以证明设备吞吐饱和。

建立“观察→瓶颈→干预→结果”的证据链：例如某处理线程满核、设备仍有余量、该队列等待增加，并且移除该环节后在相同 CPU 预算下改善，才支持软件路径瓶颈判断。

null backend／null bdev 可用于测量软件上限和逐层开销，但不具有数据持久化能力，其 IOPS 不参与接近裸盘的验收。

## 6. 改造路线与选择门槛

### A. 建立现有能力的最优基线

确认本地放置、strict-local、实际 revision counter、CPU 竞争、NUMA、文件系统和 V2 前端。先量化已有能力，决定是否仍需 fork。配置差异、语义差异与代码收益分别记录。

### B. V1 低侵入优化

以 profile 为前提依次验证：

1. 单 writer 快路径：避免通用 fanout 的额外分配／goroutine／等待；保留错误转换、短写和控制器状态处理契约。
2. 减少协议对象、完成通知和缓冲分配：核实异步请求完成前的 buffer 所有权，避免 pool 复用导致数据污染。
3. 如果集中处理循环饱和，再研究分片队列／多队列；验证重叠写、flush 顺序、超时与取消，不能只增加连接数。
4. 对测量确认的非对齐、索引和锁热点优化；不把数据竞争隐藏为性能提升。

这一路线适合快速检验低风险收益，但不能预期删除一个 goroutine 就消除前端和进程边界的大部分成本。

### C. 本地单副本合并数据路径原型

若 IPC／协议处理占据主要开销，研究将本地 engine 与 replica 存储接口置于同一进程／共享运行时，使数据请求直接调用存储层，同时继续由 Longhorn 管理卷的生命周期。

原型必须复用或等价实现副本的活动层、快照链、元数据、flush 和错误处理。直接打开 head 文件会绕过这些语义，只能作为成本下界对照。

进程合并会改变故障隔离、内存生命周期和 instance-manager 管理契约。第一版采用离线 detach→切换→attach，不进行有在途 I/O 的路径热切换，也不在故障时自动回退旧路径。

### D. V2／UBLK 与本地 bdev 路径

如果前端或 Go 每 I/O 开销占主导，优先验证上游 V2／UBLK，并检查实际本地 bdev 图。仅在确实存在可移除的内部 NVMe-oF 传输层时，再研究同进程本地 bdev 直连。

比较时维持同一 RAID 硬件和 CPU 预算。SPDK 轮询、多核和内存资源的代价必须进入结果。V2 的故障处理、磁盘接入与备份能力需要按精确版本验证，不能直接宣称与 V1 等价。

### E. 写入所有权与可选能力

所有新路径均需要数据面可执行的单写者约束：attach generation／epoch、拒绝旧实例的新请求、清理或等待在途 I/O、确认旧设备无法继续写，再允许新实例接管。具体机制要覆盖实际本地设备访问路径，仅有 Kubernetes Lease 不足以阻止失联进程写盘。

保留控制平面并不意味着每个 I/O 都访问 CRD；控制器负责授权、状态和生命周期，数据面负责执行写入边界。

快照、revert、备份、扩容、discard、backing image 等逐项列出“支持／限制／未验证”。功能可分期恢复，但普通写及同步写的正确语义从第一个原型开始就必须保留。

## 7. 正确性与故障实验

性能与正确性是两道独立验收门槛：

1. 读写内容校验、并发非重叠写、重叠写的合法顺序、短写／部分失败传播、flush 前后顺序，以及超时请求的最终状态。
2. 使用带序号与 CRC 的事务记录；独立见证端只记录已经完成规定持久化屏障的事务。故障后，见证端确认的事务必须可恢复，未确认尾部允许出现或消失；重叠覆盖场景按屏障关系验证，不能要求保留被后续合法覆盖的数据。
3. 注入 engine／replica／instance-manager 进程退出、节点重启及受控掉电；`kill -9` 只能证明进程崩溃恢复，不能代替断电测试。
4. 注入磁盘满、I/O 错误、网络中断／恢复、旧实例恢复与重复 attach，验证错误反馈及 fencing。
5. 验证快照创建／读取／回滚、备份恢复、扩容、discard 与 backing image 的约定行为。discard 后读值按已声明接口契约判断，不能无条件假设为零。
6. 同时记录数据格式兼容性、升级／降级与回退路径。新格式或新模式不能由不理解它的旧 engine 自动接管。

单副本所在节点不可用会导致该卷不可用；RAID 0 成员损坏可导致整个阵列数据丢失。这是实验及产品模式固有的故障模型，路径精简不增加冗余能力。

## 8. 执行阶段、仓库归属与交付

| 阶段 | 工作 | 阶段出口 |
| --- | --- | --- |
| S0 基线冻结 | 建立专用环境，核对镜像与源码，确定数据语义和 CPU 预算 | 可复现环境清单、原始基线数据 |
| S1 分层定位 | R0–R4、strict-local、V2／UBLK 对照与 profile | 有数据支持的前两项瓶颈及置信边界 |
| S2 最小原型 | 从 B／C／D 选择证据最强的方向，单项 A/B | 补丁、profile 变化、同口径收益或负结果 |
| S3 正确性 | 校验、故障注入、屏障与 fencing 测试 | 可复现测试和未通过项；不以速度抵消正确性失败 |
| S4 VM 与竞争 | Harvester VM、多卷、后台操作及长稳测试 | VM 端收益、资源成本、支持范围与接入建议 |

主要代码归属：V1 engine 负责 controller、dataconn、replica；manager／instance-manager 负责进程启动和生命周期；V2 路径涉及 SPDK engine 及前端依赖。CSI 与 Harvester 负责能力暴露、StorageClass、VM 接入及打包验证。以锁定版本的实际仓库和依赖结构为准。

本 Harvester 仓库本次只增加研究文档。修改 `upgrade_controller.go` 无法直接改善上述数据路径。未来实现应在拥有对应 I/O 路径的仓库开发，Harvester 集成在原型通过验证后进行。

每次实验归档环境 manifest、镜像／源码哈希、fio job 与 JSON、profile、原始监测序列、补丁和正确性报告。汇总表必须包含原版、候选版本、R0、R1、实际队列、CPU 预算、吞吐、尾延迟、持久化模式和功能状态，不填写没有实测依据的结果。

优先决策：先量化 V1 strict-local 和 V2／UBLK 的现有上限。若单 writer 的分配／调度仅占小部分，停止围绕它微调；若跨进程协议或前端占主导，再投入合并本地路径。研究价值在于解释并消除可证明的成本，同时保留数据保证。

## 9. 参考资料

- [Longhorn 官方性能调查](https://github.com/longhorn/longhorn/wiki/Longhorn-Performance-Investigation)：已有前端／分层性能研究，数据仅适用于原测试条件。
- [V1 1.11.2 单 writer 分发](https://github.com/longhorn/longhorn-engine/blob/v1.11.2/pkg/controller/multi_writer_at.go)。
- [V1 1.11.2 dataconn client](https://github.com/longhorn/longhorn-engine/blob/v1.11.2/pkg/dataconn/client.go) 与 [server](https://github.com/longhorn/longhorn-engine/blob/v1.11.2/pkg/dataconn/server.go)。
- [V1 1.11.2 remote backend](https://github.com/longhorn/longhorn-engine/blob/v1.11.2/pkg/backend/remote/remote.go)。
- [V1 1.11.2 replica 写入](https://github.com/longhorn/longhorn-engine/blob/v1.11.2/pkg/replica/replica.go)、[diff disk](https://github.com/longhorn/longhorn-engine/blob/v1.11.2/pkg/replica/diff_disk.go)、[revision counter](https://github.com/longhorn/longhorn-engine/blob/v1.11.2/pkg/replica/revision_counter.go)。
- [Manager 1.11.2 strict-local 协议选择](https://github.com/longhorn/longhorn-manager/blob/v1.11.2/engineapi/instance_manager.go)。
- [V2 1.11.2 性能文档](https://longhorn.io/docs/1.11.2/v2-data-engine/performance/)、[UBLK 前端](https://longhorn.io/docs/1.11.2/v2-data-engine/features/ublk-frontend-support/)、[磁盘接入](https://longhorn.io/docs/1.11.2/v2-data-engine/features/node-disk-support/)。

上游 wiki 和网站可更新；正式实验需归档所依据的文档内容和源码 commit，避免把后来版本的能力带入基线结论。
