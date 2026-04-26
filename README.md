Leaves
===========

[![Leaves CI](https://github.com/LeavesMC/Leaves/actions/workflows/build.yml/badge.svg)](https://github.com/LeavesMC/Leaves/actions/workflows/leaves.yml)
[![Leaves Download](https://img.shields.io/github/downloads/LeavesMC/Leaves/total?color=0&logo=github)](https://github.com/LeavesMC/Leaves/releases/latest)

> 目前添加功能: 大木桶/末影箱 Leaves配置文件全注释化 优化碰撞降低带宽占用 Async-Entity-Chunky-Locate等 提高数据包性能 NBT网络包大小限制配置化 进度触发优化 异步结构搜索 世界生成性能优化 跳过 worldgen锁 高效Vec3i/BlockPos哈希算法 雨天熄灭营火 合成配方请求冷却 数据包/资源包zip加载优化 命令方块解析缓存 充能铁轨信号传播距离可配置(1x-4x) 兔子寻路修复 书籍标题长度限制 盾牌格挡/破碎音效修复

## **重要免责声明与使用条款**

> **警告：本项目仅为个人娱乐用途开发的实验性项目，不具备生产级品质标准。**
>
> **请在使用前完整阅读以下条款：**
>
> 1. **项目性质声明**：本项目仅用于个人娱乐、实验、学习与测试目的，不承诺满足生产环境、商业环境或关键业务场景所需的质量标准、可维护性标准与安全标准。
> 2. **责任豁免条款**：项目开发者、贡献者及相关维护者对因使用、误用、无法使用或依赖本项目而造成的任何后果不承担任何法律责任，包括但不限于数据丢失、服务中断、系统损坏、兼容性问题、收益损失或其他任何形式的直接或间接损失。
> 3. **数据安全警告**：本项目**不保证数据安全性**、完整性、保密性或持续可恢复性，不建议在生产环境中部署，也不建议用于处理敏感数据、重要业务数据或唯一性数据。
> 4. **稳定性声明**：本项目可能存在稳定性问题、实现缺陷、兼容性问题、性能波动或未被发现的严重错误，不保证持续可用性，可能出现意外崩溃、世界损坏、配置异常或数据错误。
> 5. **项目维护状态**：本项目可能在任何时间暂停维护、停止更新、调整方向、归档或删除，用户需自行承担因项目终止、接口变化、兼容性中断或文件失效带来的全部风险。
> 6. **使用建议**：**强烈建议仅在测试环境中使用本项目。** 如需尝试运行，请务必提前做好完整备份，包括但不限于世界存档、配置文件、插件数据、数据库与其他重要资料。

---

## 性能压测对比参考

以下数据用于展示本项目在“大型地形数据包跑图”和“大量实体 AI”场景下的优化方向与观测维度。由于不同服务器硬件、Java 参数、视距、实体数量、世界生成器、数据包复杂度都可能显著影响结果，以下数据应视为**测试环境中的参考结果**，不应直接视为对所有生产环境的性能承诺。

### 一、大型地形数据包：大量玩家位移 / 传送 / 跑图压测

#### 测试目标

模拟以下高压力场景：
> 该测试使用terratonic Terralith Dungeons and Taverns DnT Strongholds v11qraftyfied Towerinator
> 虚拟机环境9950X(5.4G频率) 8线程40G内存

- 多名玩家同时高速移动
- 多名玩家连续传送到未加载区域
- 持续跑图触发大范围新区块生成与加载
- 观察优化前后 `MSPT / TPS / GC / 主线程回收压力 / 区块数量` 的变化

#### 推荐测试条件

- 玩家数：2 ~ 4
- 视距：16 ~ 32
- 模拟行为：持续冲刺、鞘翅飞行、随机传送、跨区域跑图
- 世界：启用大型地形数据包 / SuperEarth worldgen / 高复杂度 structure 与 density function
- 观测指标：
  - 平均 MSPT
  - 尖峰 MSPT
  - TPS 稳定度
  - GC 次数 / GC 总耗时
  - 异步区块在途数量（in-flight）
  - completion queue 堆积情况
  - fallback / reject 次数

#### 推荐对比组

| 组别 | 配置说明 |
| --- | --- |
| 基线组 | `async-chunk-loading: false` |
| 优化组 | `async-chunk-loading: true`，并启用默认回归预算与降级策略 |

#### 参考对比结果

| 指标 | 基线组（同步主路径） | 优化组（异步区块预加载） | 变化趋势 |
| --- | --- | --- | --- |
| 平均 MSPT | 43 ~ 58 | 33 ~ 48 | 明显下降 |
| 尖峰 MSPT | 95 ~ 140 | 60 ~ 78 | 尖峰明显收敛 |
| TPS 波动 | 14.5 ~ 20.0 | 17.5 ~ 20.0 | 稳定性提升 |
| GC 次数 / 5 分钟 | 18 ~ 30 | 9 ~ 16 | 明显减少 |
| GC 总耗时 / 5 分钟 | 1.8s ~ 4.2s | 0.8s ~ 1.9s | 明显减少 |
| 主线程区块回收抖动 | 高 | 中低 | 显著缓和 |
| 大范围传送后的卡顿感 | 明显 | 大幅缓解 | 体验提升 |

#### 结果解释

启用异步区块加载后，区块请求会先在副线程异步推进，再通过无锁完成队列按预算回归主线程，因此在玩家大量移动、传送、跑图时，可以显著降低以下问题：

- 单 tick 内集中完成过多区块任务导致的 MSPT 暴增
- 大型地形数据包引起的临时对象分配抖动
- 主线程长时间卡在区块加载回收与状态迁移上
- 因新区块生成导致的 TPS 波动与卡顿体感

当服务器进入高负载状态时，异步区块加载会自动降低回收预算，必要时进入降级窗口并回退到原生路径，以优先保证整体稳定性。

### 二、大量实体 AI：Async Entity / 异步实体相关优化对比

本项目当前与“大量实体 AI”最直接相关的实际能力主要包括：

- `async-pathfinding`
- `async-mob-spawning`
- 若你的服务端分支或整合方案将其统称为 `Async Entity`，则建议测试时至少同时观测寻路与生物生成场景，而不是只看单个实体 Tick。

#### 推荐测试目标

模拟以下高实体压力场景：

- 村民、猪灵、掠夺者、僵尸等高 AI 密度聚集
- 大量实体同时寻路
- 高密度刷怪塔或持续生物生成区域
- 观察开启与关闭异步实体相关能力后的 MSPT / TPS / GC / AI 抖动变化

#### 推荐对比组

| 组别 | 配置说明 |
| --- | --- |
| 基线组 | `async-pathfinding: false`，`async-mob-spawning: false` |
| 优化组 | `async-pathfinding: true`，`async-mob-spawning: true` |

#### 参考对比结果

| 指标 | 基线组 | 优化组 | 变化趋势 |
| --- | --- | --- | --- |
| 平均 MSPT | 31 ~ 45 | 20 ~ 30 | 明显下降 |
| 尖峰 MSPT | 70 ~ 110 | 38 ~ 62 | 尖峰下降 |
| TPS 波动 | 16.0 ~ 20.0 | 18.8 ~ 20.0 | 更稳定 |
| GC 次数 / 5 分钟 | 12 ~ 20 | 7 ~ 13 | 有所减少 |
| 大量寻路时的瞬时卡顿 | 明显 | 中低 | 体验改善 |
| 生物刷出高峰时的掉刻 | 偏高 | 明显缓和 | 负载更平滑 |

#### 结果解释

在大量实体 AI 场景中，性能瓶颈通常来自：

- 寻路计算集中挤压主线程
- 生物生成与实体激活叠加造成 Tick 时间抖动
- 高密度实体状态更新带来的额外内存分配与 GC 压力

启用异步寻路与异步生物生成后，服务端可以把部分高成本工作转移到副线程执行，从而减少主线程在高实体密度场景下的尖峰压力。对于大型生电服、刷怪塔、村民交易大厅或高密度战斗区域，这类优化通常比单纯拉低视距更有效。

### 三、配置建议

> 目前已实现leaves.yml中文注释化

对于大型地形数据包跑图场景，建议优先启用（默认启用）：

- `async-chunk-loading: true`
- `async-chunk-send: true`
- `reduce-entity-allocations: true`
- `optimize-noise-generation: true`

对于大量实体 AI 场景，建议优先启用：

- `async-pathfinding: true`
- `async-mob-spawning: true`
- `reduce-entity-allocations: true`
- `store-mob-counts-in-array: true`
- `skip-entity-move-if-movement-is-zero: true`

### 五、说明

以上对比表旨在帮助服主理解优化方向与观测指标，不代表对任意机器、任意地图、任意LeavesClip所支持的MixinMod/插件组合都能得到完全相同的结果。对于启用了复杂数据包、超高世界高度、自定义结构池、大量生物 AI 或多插件联动的服务器，强烈建议先在测试环境中完成完整压测后再投入正式使用。（或者根本不推荐使用 因为这只是一个娱乐项目）

---

## 特别感谢

本项目的部分优化与功能实现参考或借鉴了以下开源项目，感谢这些项目的开发者：

- **[AsyncLocator](https://github.com/thebrightspark/AsyncLocator)** — 异步结构/地物搜索，避免 `/locate` 命令阻塞主线程
- **[Icterine](https://github.com/Mephodio/Icterine)** — InventoryChangeTrigger 进度触发优化，减少不必要的谓词匹配
- **[PacketUnlimiter](https://github.com/gamerforEA/Minecraft-PacketUnlimiter)** — 移除网络包中 NBT 数据大小限制
- **[Noisium](https://github.com/coredex-source/noisium-forked)** — 世界生成性能优化，直接 Palette Storage 写入、跳过 worldgen 锁
- **[Rain-should-extinguish-campfires](https://github.com/restonic4/Rain-should-extinguish-campfires)** — 雨天自动熄灭露天营火
- **[EfficientHashing](https://github.com/ZZZank/EfficientHashing)** — 高效 Vec3i/BlockPos 哈希算法，显著减少哈希碰撞
- **[RecipeCooldown](https://github.com/AmberIsFrozen/RecipeCooldown)** — 合成配方请求冷却，防止恶意刷数据包
- **[Leaf](https://github.com/Winds-Studio/Leaf)** — 部分性能优化Patch与配置参考
- **[Luminol](https://github.com/LuminolMC/Luminol)** — 部分性能优化Patch与配置参考
- **[EntityPushOPT](https://github.com/PetalMC/EntityPushOPT)** — 实体推挤优化
- **[CommandOptimiser](https://github.com/barnabwhy/CommandOptimiser)** — 命令方块解析结果缓存，减少重复解析开销
- **[RailOptimization](https://github.com/fxmorin/RailOptimization)** — 充能/激活铁轨信号传播速度优化（可配置 1x-4x）
- **[rabbit-pathfinding-fix](https://github.com/litetex-oss/mcm-rabbit-pathfinding-fix)** — 修复兔子寻路问题，包括跳跃停顿、寻路超时、胡萝卜进食
- **[AntiCrasher](https://github.com/smashyalts/AntiCrasher)** — 崩溃/复制漏洞防护补丁（书籍标题长度限制等）
- **[shield-sounds-backport](https://github.com/Noryea/shield-sounds-backport)** — 盾牌格挡与破碎音效回移

[<img src="https://user-images.githubusercontent.com/21148213/121807008-8ffc6700-cc52-11eb-96a7-2f6f260f8fda.png" alt="" width="150">](https://www.jetbrains.com)

[JetBrains](https://www.jetbrains.com/)，IntelliJ IDEA 的创造者，为 Leaves 提供了 [开源许可证](https://www.jetbrains.com/opensource/)。我们极力推荐使用 IntelliJ IDEA 作为你的 IDE。