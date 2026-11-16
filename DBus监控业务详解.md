# DBus 监控业务详解：这个仓库在 OpenBMC 里到底扮演什么角色

> 面向 Java 背景、有 openUBMC 经历、正在学 OpenBMC 的你。
> 本文**不重复讲代码结构**（那些看 `ONBOARDING.zh-CN.md` 和配套《DBus监控机制讲解.md》），而是回答三个业务问题：
> ① 真实服务器里它替你办什么事？② 为什么非得有它、别的仓库不能干吗？③ 它和你已知的另外几个仓库怎么分工？
>
> 讲法：先给你一个能想象的真实场景，再点出背后的业务概念，最后告诉你"对应代码里的哪一段"。
> 📌 路径说明：本文所有代码都在 `phosphor-dbus-monitor/src/` 下，为节省篇幅正文中只写文件名或 `src/xxx`，
> 带行号的引用形如 `src/count.hpp:36`，构建配置在仓库根的 `meson.build`。

---

## 0. 一句话说清业务

把你熟悉的 openUBMC 告警心智搬过来：这个仓库就是 OpenBMC 里的**"全总线观察员 + 规则触发器"**。

它自己**不产生任何业务数据**，也不持有服务器的"电源状态""传感器读数"这类权威状态。
它只做一件事：**盯着总线上别人报出来的属性和信号，发现符合你事先写好的规则，就替你做出反应**——
比如"温度高了就记 error log"、"风扇转速归零就发 journal 日志"、"某对象出现了就调它的方法"。

> **业务一句话：你写 YAML 声明"什么情况该做什么"，它 7×24 盯着 D-Bus，情况一出现就自动执行动作。**
> 这就是 `ONBOARDING.zh-CN.md:18` 说的"声明式监控 —— 你写 YAML，它替你盯 D-Bus"。

---

## 1. 先讲个真实场景：机房里那批温度传感器的"三高告警"

把这个过程在脑子里过一遍，后面所有业务概念都从这个故事里长出来。
这不是我编的例子——它就写在仓库自带的示例配置 `src/example/example.yaml:207-238` 里。

```
① 机房里有一排温度传感器，每个传感器都是一个 D-Bus 对象：
   /xyz/openbmc_project/testing/inst1 ... inst4
   它们都暴露 xyz.openbmc_project.Sensor.Value 接口的 Value 属性
   （单位：摄氏度，这是 example.yaml:39-44 里定义的 property group）

② 运维提了一条需求（业务规则）：
   "这 4 个传感器里，只要有任意 3 个读数 >= 115 度，
    就说明机箱真的过热了，要立刻记一条 error log 告警，
    并且把'是哪几个传感器超限'一起写进日志。"

③ 这条规则被翻译成一段 YAML（不是写 C++！）放进机器层配置：
   - condition: count，op: ">=" bound: 115（单个传感器判定：读数 >= 115）
   - countop: ">=" countbound: 3（数量判定：至少 3 个达标）
   - oneshot: true（不要在持续高温期间每秒刷一条告警）
   - callback: elog（达标后记 error log）
   见 src/example/example.yaml:229-238

④ 构建镜像时，pdmgen.py 把这段 YAML "编译"成 C++ 源码 generated.hpp
   （构建期做一次，运行时不再解析 YAML —— src/meson.build:24-44）

⑤ BMC 运行起来后，本进程：
   - 启动时先 GetAll 把所有传感器当前读数拉进缓存（开机全量检查，main.cpp:43-46）
   - 之后只"听"各传感器的 PropertiesChanged 信号（propertywatchimpl.hpp:72-79）
   - 每来一个新温度值：更新缓存 → 跑一遍两遍求值（count.hpp:54-90）
     → 一旦"3 个 >= 115"成立 → 触发 elog 动作，日志里自动带上超限传感器的路径和值
     （elog.hpp:184-206 的 captureMetadata 从求值结果槽里抓出"哪些属性达标"）
```

**这就是本仓库的业务主线**：第③步"写规则" + 第⑤步"盯总线、触发动作"。
它不像电源控制那样亲自动手拉引脚，而是像**质检员**：盯着仪表、对照标准、超标了就填单上报。

> 📌 这个"温度三高"故事就是理解全部业务的钥匙。后面讲的声明式、观察枢纽、两遍求值、oneshot，
> 全是为了把第③步那条规则准确、不重复、不漏报地执行出来。

---

## 2. 为什么非得让一个独立进程干这事（业务价值）

你可能会问：读个传感器值、判断一下阈值，sensor 服务自己不能做吗？为什么要单独一个进程盯着全总线？
三个真实价值，用你熟悉的 openUBMC 类比着看：

| 业务价值 | 场景 | 没有它会怎样 |
|---|---|---|
| **规则集中、跨服务**（最重要） | 告警条件常常横跨多个服务：温度来自 sensor、开机状态来自 state-manager、日志要交给 logging | 每个服务各写一套判断，规则散落、口径不一，新增一条规则要改好几个服务 |
| **带外兜底** | 主机 OS 死了、业务服务没起来，但 BMC 上的 D-Bus 还活着 | 没人盯那些"该来没来"的信号，故障无人上报 |
| **配置驱动、免重编译** | 机器 A 要 3 个传感器超限才告警，机器 B 要 5 个 | 得为每种机型改 C++、重新编译；现在只改 YAML，重新生成即可（`src/meson.build:24-44`） |

**一句话**：把"什么时候该做什么"这种**策略**从各个**业务服务**里抽出来，集中到一个观察员身上。
对比你熟的 Java：这就是把散落在各处的 `if (temp > 115) {...}` 抽成一套**规则引擎 / 观察者模式**——
业务服务只管发布事实（属性变化），规则集中声明在配置里，观察员负责匹配和动作。
OpenBMC 里这类"配置驱动、"的思想你已经在 systemd 的 target/service 里见过了，这里是它在监控领域的翻版。

> 📌 注意区分：它**不是**用来替代 sensor 采样的（数据还是 sensor 服务报的），
> 也**不是**用来替代 phosphor-logging 的（真正的 error log 还是 logging 建的，见 `src/elog.hpp:74-78`
> 里 `phosphor::logging::report<T>` 的调用）。它只负责"判断 + 触发"。

---

## 3. 必懂的业务概念（先场景，后术语）

这些是你读懂代码缺的那块"领域知识"。每个都先说它在故事里是什么角色，再给术语，最后给代码位置。

### 3.1 声明式监控：写"规则"而不是写"流程"

回到第 1 节的故事：你并没有写"先读 inst1、再读 inst2、然后比较、最后记日志"这样的**流程**，
你只声明了**事实**："满足 count>=3 且 value>=115 时，执行 elog"。
至于"怎么读、怎么比、什么时候触发"，全由框架决定。

- **术语**：声明式（declarative） vs 命令式（imperative）。
- **Java 类比**：命令式像你手写的一段 `for` 循环；声明式像 SQL 的 `SELECT ... WHERE` 或 Spring 的 `@EventListener`——
  你只说"关心什么条件、条件到了干什么"，不关心底层怎么轮询、怎么调度。
- **落地**：YAML 的几大类指令——`conditions`（条件）、`callbacks`（动作）、`watch`（监听）、`group`（分组）
  在 `src/pdmgen.py:1249-1265` 的 `Everything.__init__` 里被接收；
  整条"YAML → C++"的生成流程写在 `src/pdmgen.py:3-16` 的模块注释里。

> 📌 业务上最关键的一点：**改监控规则 = 改文本配置 + 重新生成**，不碰业务逻辑代码。
> 这是它存在的根本理由，也是你以后收到"加一条告警规则"需求时该走的正确路径。

### 3.2 观察枢纽：它是"全总线"的耳朵和眼睛

故事里那 4 个传感器的属性变化，并不是发给本进程的。它们只是往 D-Bus 上广播了一个
`PropertiesChanged` 信号。本进程通过"订阅规则"（match）把自己的回调挂到总线上，谁说话它都能听见。

- **术语**：D-Bus 的 **signal / match** 机制——发布方广播，订阅方按规则接收，两者互不认识（松耦合）。
- **业务含义**：它是**观察枢纽（observer hub）**，一个进程同时监听全系统成百上千个属性，
  而不需要每个业务服务来"通知"它。
- **Java 类比**：观察者模式（Observer）/ 事件总线（EventBus）。传感器是被观察者（Subject），
  本进程是观察者（Observer），D-Bus 就是那条 EventBus。
- **落地**：match 在 `PropertyWatch::start()` 里注册——
  `InterfacesAdded` 的 match 在 `src/propertywatchimpl.hpp:44-46`，
  `PropertiesChanged` 的 match 在 `src/propertywatchimpl.hpp:72-79`；
  底层 `addMatch` 把订阅对象存在一个静态 vector 里保活（`src/sdbusplus.hpp:90-94`）。

> 📌 正因为它是"旁听"而非"被通知"，所以它**不持有业务状态**、也不影响被观察服务的正常工作——
> 你把它理解为装在总线上的一只"监听探头"。SVG 图里把它标为"纯消费者 + 触发者（不持有业务状态）"。

### 3.3 条件与动作：规则的两半

一条规则 = 一个**条件**（什么时候算数）+ 一组**动作**（算数了干什么）。

| 概念 | 在故事里的角色 | 术语 / 代码 |
|---|---|---|
| **条件 Condition** | "3 个传感器 >= 115 度"这个判断 | `Conditional` 抽象，`src/callback.hpp:59-71`；计数条件 `CountCondition`，`src/count.hpp:36` |
| **动作 Callback** | "记一条 error log"这个反应 | `Callback` 抽象，`src/callback.hpp:24-52` |
| **绑定** | "条件成立就触发动作"这层胶水 | `ConditionalCallback`，`src/callback.hpp:164-192`，核心判断在 `src/callback.hpp:178-184` |
| **监听 Watch** | "盯着哪些对象的哪些属性" | `PropertyWatch`，`src/propertywatch.hpp:32`；`Watch` 接口在 `src/watch.hpp:26` |

**Java 类比**：`ConditionalCallback` 就是一个带 `if` 的装饰器（Decorator）——
它包住一个真正的 Callback，先把条件跑一遍，条件过了才调用背后的 Callback（`src/callback.hpp:180-182`）。

### 3.4 两种"告警出口"：error log vs Events 对象

故事第⑤步说"记 error log"，但本仓库其实有**两个**出口，业务上别混：

| 出口 | 业务含义 | 谁来看 | 代码 |
|---|---|---|---|
| **error log（elog）** | 正式的硬件/系统故障记录，走 phosphor-logging，会进 Redfish 的日志清单 | 运维在 Web/Redfish 的"事件日志"里查 | `src/elog.hpp:46-93`，最终调 `phosphor::logging::report`（`src/elog.hpp:78`） |
| **journal** | 轻量系统日志，带 MESSAGE_ID | 开发者/运维 `journalctl` 看 | `src/journal.hpp` |
| **Events 对象** | 本仓库自建的轻量事件，按类别在 `xyz.openbmc_project.Events` 下枚举 | 界面/客户端按事件类别枚举 | `src/event_manager.cpp:31-86`（`create`）、`src/event_manager.cpp:88-127`（`restore`） |

> 📌 区分要点：error log 是"报给 logging 的正式工单"，Events 对象是"本进程自己记的小台账"。
> 两者并存，容量封顶（`MAX_EVENTS=20`，见根 `meson.build:23`，满了删最老，`src/event_manager.cpp:70-78`）。

### 3.5 两个上下文：开机全量检查 vs 信号增量触发

故事第⑤步其实有两个时机：

- **开机时**：进程刚起来，先把所有被监控属性的**当前值**拉一遍、跑一遍规则——
  万一传感器在 BMC 启动前就超限了，不能因为没"听见"变化就漏报。
- **运行时**：之后每次属性变化，增量跑一遍规则。

框架用 `Context` 这个枚举区分这两种情况（`src/data_types.hpp:38-42`）：

```
Context::START   —— 启动时全量检查（main.cpp:43-46 对每个 watch 调一次）
Context::SIGNAL  —— 属性变化触发（propertywatchimpl.hpp:178）
```

**业务意义**：有些动作**只在信号时做、开机时不做**。
比如 `ElogWithMetadataCapture` 在 `Context::START` 时直接返回不记日志（`src/elog.hpp:158-162`），
避免"开机时传感器恰好超限"就在没发生新变化的情况下刷一条日志。
写规则时你要意识到"我这条回调会在两种上下文里都被调用"。

---

## 4. 一条规则的"业务生命周期"

把第 1 节那条温度规则，从你写下它到它生效，按时间顺序走一遍。这张表是你以后做需求时的"操作地图"：

| 阶段 | 谁在做 | 发生什么 | 代码位置 |
|---|---|---|---|
| ① 声明 | 你（在机器层 YAML / bbappend） | 写 path group、property group、count condition、elog callback、property watch | `src/example/example.yaml:3-44, 75-94, 207-238` |
| ② 生成 | 构建系统（pdmgen.py） | 扫描所有 YAML → 解析引用 → 渲染成 C++ `generated.hpp` | `src/pdmgen.py:1218-1244`（load）、`src/pdmgen.py:1269-1282`（generate_cpp）、`src/meson.build:24-44` |
| ③ 启动 | 本进程 main() | 挂 ObjectManager、恢复事件、逐个 `start()` 注册 match、逐个跑一次 `callback(START)`、进事件循环 | `src/main.cpp:28-59` |
| ④ 监听 | PropertyWatch | 启动时 GetAll 拉满缓存；运行时收 PropertiesChanged/InterfacesAdded 更新缓存 | `src/propertywatchimpl.hpp:27-112, 131-140` |
| ⑤ 求值 | ConditionalCallback + CountCondition | 缓存更新即求值：两遍判定 + oneshot 锁存 | `src/callback.hpp:178-184`、`src/count.hpp:54-90` |
| ⑥ 动作 | Callback 动作链 | 条件成立 → 按 graph 找到回调 → elog/journal/建事件 | `src/callback.hpp:147-153`、`src/elog.hpp` |

> 📌 业务上你要建立的直觉：**② 是构建期，③④⑤⑥ 是运行期**。运行时没有任何 YAML 解析——
> 规则已经被"编译"进了二进制。这正是它高效、也正因为它"改规则要重新构建"的原因。

---

## 5. 真实业务场景速查（现象 → 规则怎么配）

把常见运维需求翻译成"该配哪类条件/动作"，这是你接到需求时最有用的一张表：

| 业务场景 | 规则该怎么配 | 用到的机制 |
|---|---|---|
| N 个传感器里至少 M 个越界才告警 | count condition：`op/bound` 定单项阈值，`countop/countbound` 定数量阈值 | `src/count.hpp:54-90` 两遍求值 |
| 按"中位数"判断一组传感器整体漂移 | median condition，配置类似 count，但比较中位数 | `src/pdmgen.py:848-868` |
| 持续告警但不要刷屏 | `oneshot: true`，条件重新变 false 才允许再触发 | `src/count.hpp:83-89` |
| 告警要带"哪个传感器坏了"的明细 | `ElogWithMetadataCapture`，从结果槽抓达标的属性拼元数据 | `src/elog.hpp:141-206` |
| 抖动信号要稳定一段时间才算数 | `defer: 1000us` 延迟回调，条件持续满足才触发 | `src/callback.hpp:206-271`、`src/example/example.yaml:240-259` |
| 属性落在合法区间才参与判断 | 给 watch 配 `filters`，不符的值被清出缓存 | `src/example/example.yaml:68-72`、`src/filters.hpp:41-83` |
| 某 D-Bus 对象"出现"时触发（而非属性值变化） | path watch 监听 `InterfacesAdded` | `src/propertywatchimpl.hpp:44-46`、`src/pdmgen.py:719-724` |
| 开机时就要检查一遍当前状态 | 不设 `ignore_start_callback`，让 START 上下文进来 | `src/propertywatchimpl.hpp:119-122`、`src/example/example.yaml:73` |
| 条件成立后调用别的服务的方法 | method callback，必要时经 mapper 反查服务名 | `src/sdbusplus.hpp:97-120` |

---

## 6. 和另外 5 个仓库的分工（边界别混）

你本地 `E:/openbmc/` 下还有几个相关仓库。用你熟悉的微服务类比，把边界划清：

```
传感器服务（sensor)  ──报出读数──▶  D-Bus 总线
                                      │
phosphor-dbus-monitor ◀──旁听──────────┤  ← 本仓库：观察规则、判定、触发
   （观察枢纽 / 规则层）               │
        │ 触发                        │
        ├─▶ phosphor-logging    正式故障工单（elog）
        ├─▶ phosphor-sel-logger 系统事件记录（SEL，本质也是 logging 的一种落盘形式）
        ├─▶ phosphor-state-manager  主机/机箱状态（本仓库只读它、不写它）
        └─▶ x86-power-control  电源动作（本仓库一般只调方法，不动手拉线）
```

逐个说清谁干什么、和本仓库的关系：

| 仓库 | 它的职责 | 和本仓库的关系 |
|---|---|---|
| **phosphor-dbus-monitor**（本仓库） | 观察 + 判定 + 触发；规则层 | 主角。自己不产数据、不持有权威状态 |
| **phosphor-logging** | 创建、存储、对外暴露正式的 error log（Redfish LogEntry） | 下游：本仓库通过 `phosphor::logging::report<T>` 委托它记日志（`src/elog.hpp:78`） |
| **phosphor-sel-logger** | 把日志落成 SEL（System Event Log），供 IPMI 查 | 同属"日志家族"；本仓库的 elog 最终可经 logging 进入 SEL 体系 |
| **phosphor-state-manager** | 维护主机/机箱/BMC 的权威状态机（Running/Off…），靠 systemd target 编排 | 上游数据源：本仓库监听它的属性变化当条件，但**不写**它的状态 |
| **x86-power-control** | 真正拉电源脚、盯 PWROK、维护电源状态（"手"） | 下游/旁听：本仓库可监听电源状态属性、也可通过 method 触发动作，但不亲自拉线 |

**一句话记忆**：
> state-manager 是"大脑"（决定状态）、x86-power-control 是"手"（动手操作硬件）、
> phosphor-logging / sel-logger 是"账本"（记录故障）、**本仓库是"eyes & rules"（盯着、判定、按下报警按钮）**。
> 它把"大脑/手/账本"串成一条自动化流水线，但自己不替代其中任何一个。

> 📌 对 openUBMC 经验的迁移提示：如果你以前用过"告警规则配置 + 告警服务"那套，
> 本仓库大致相当于那个"告警服务 + 规则引擎"，而 sensor 服务相当于上报数据的组件。
> 区别是 OpenBMC 把它做成了**构建期生成 C++**，而不是运行时加载规则。

---

## 7. 常见业务故障场景（现象 → 原因 → 看哪里）

从业务现象反推，帮你把"配置/代码"和"真实故障"对上，当作排错速查表：

| 业务现象 | 可能原因 | 该查哪里 |
|---|---|---|
| 传感器已超限，却没告警 | 传感器对象路径/接口名配错，match 没匹配上 | `src/example/example.yaml:39-44` 的 property group；`src/propertywatchimpl.hpp:72-79` 的 match |
| 告警一直刷屏 | `oneshot` 没开，持续越界每次变化都触发 | `src/count.hpp:83-89`；`src/example/example.yaml:238` |
| 只在信号来时才报，开机时就超温却没报 | `ignore_start_callback: true` 跳过了 START 检查 | `src/propertywatchimpl.hpp:119-122`；`src/example/example.yaml:73` |
| 告警日志里看不到"哪个传感器坏了" | 用了普通 elog，没用带 metadata capture 的版本 | `src/elog.hpp:141-206` |
| 改了 YAML 但告警行为没变 | 没重新跑生成 / 没重新构建镜像 | `src/meson.build:24-44`；生成物是 `generated.hpp` |
| 抖动信号反复告警 | 没配 `defer`，瞬时满足就触发 | `src/callback.hpp:206-271`；`src/example/example.yaml:252-259` |
| 事件对象越攒越多 | 达到 `MAX_EVENTS`(20) 才删最老，短时间内大量事件 | 根 `meson.build:23`；`src/event_manager.cpp:70-78` |
| 想确认进程在不在Bus上 | 看 `xyz.openbmc_project.Events` 这个 bus name | `src/main.cpp:57`；根 `meson.build:16` |

---

## 8. 一页速记（业务版）

- **它管什么**：盯着 D-Bus 上的属性/信号，发现符合 YAML 规则就自动记日志/建事件/调方法。
- **它不管什么**：不采集传感器数据、不持有权威状态、不建 error log（委托给 logging）。
- **业务主线**：写规则(conditions+callbacks+watch) → 构建期生成 C++ → 运行时旁听总线 → 求值 → 触发动作。
- **两个上下文**：START（开机全量检查，防漏报）+ SIGNAL（信号增量触发）。
- **两个告警出口**：正式 error log（logging）+ 本进程 Events 对象（轻量台账）。
- **配置驱动**：改规则 = 改 YAML + 重新生成，不碰 C++。这是它存在的根本理由。
- **分工记忆**：大脑=state-manager，手=x86-power-control，账本=logging/sel-logger，**眼睛与规则=本仓库**。
- **Java 类比**：观察者模式/EventBus + 规则引擎 + 装饰器（条件包住动作）。

> 配套：代码结构看 `ONBOARDING.zh-CN.md`，YAML 怎么变成 C++、运行时链路看《DBus监控机制讲解.md》，
> 业务流程可视化看 `业务流程交互图.svg`。
