# phosphor-dbus-monitor 代码导读（OpenBMC 新人向）

> 面向刚从其他领域转入 OpenBMC / openubmc 开发的同学。
> 阅读本文前建议先了解：D-Bus 基本概念（bus name / object path / interface / signal / property）、
> OpenBMC 的 phosphor-objmgr（mapper）机制。

## 1. 这个仓库是干什么的

phosphor-dbus-monitor（PDM）是一个**事件监控框架**：通过 YAML 配置声明
"监视 D-Bus 上哪些属性变化，满足什么条件时执行什么动作"，构建时由代码生成器
`pdmgen.py` 把 YAML 编译成 C++ 源码（`generated.hpp`），运行时无需解析配置。

典型用途：当某传感器的值越界、某属性达到某状态时，自动：
- 创建 phosphor-logging 的 error log / journal 日志；
- 在 `xyz.openbmc_project.Events` 下创建事件 D-Bus 对象；
- 调用其他 D-Bus 方法（如控制某个服务）。

核心思想一句话：**"声明式监控" —— 你写 YAML，它替你盯 D-Bus。**

## 2. 目录结构与关键文件

```
src/
├── main.cpp               # 入口：起 bus、恢复事件、启动所有 watch、进事件循环
├── sdbusplus.hpp          # SDBusPlus：对 sdbusplus 的静态封装（总线/方法调用/addMatch）
├── data_types.hpp         # 基础类型别名（PropertyIndex、BusNameToPathMap 等）
├── watch.hpp              # Watch 抽象接口（start / callback）
├── propertywatch.hpp/.cpp # PropertyWatch 家族：监听 PropertiesChanged / InterfacesAdded
├── pathwatch.hpp/.cpp     # PathWatch：监听 mapper 的 InterfacesAdded（对象出现）
├── callback.hpp           # Callback / Conditional / ConditionalCallback / DeferrableCallback
├── filters.hpp            # 从 property 数据流中抽取/过滤字段
├── count.hpp / median.hpp # 聚合类条件：计数、中位数触发
├── method.hpp             # 回调动作：调用远端 D-Bus 方法
├── elog.hpp/.cpp          # 动作：创建 phosphor-logging error log
├── journal.hpp/.cpp       # 动作：写 systemd journal
├── event.hpp / event_entry.hpp / event_manager.* / event_serialize.*
│                          # 事件对象管理：在 D-Bus 上创建/持久化/恢复 Events 对象
├── format.hpp             # 日志消息格式化
└── pdmgen.py              # YAML → C++ 代码生成器（构建期运行）
```

测试在 `src/test/`（每个生成器都有对应的 gtest），示例配置在 `src/example/`。

## 3. 运行时架构（数据怎么流动）

```
D-Bus signal (PropertiesChanged / InterfacesAdded)
        │  addMatch 注册的 match 回调
        ▼
PropertyWatch / PathWatch  （Watch 子类，main.cpp 里逐个 start()）
        │  解析 message，更新 PropertyIndex 里的属性缓存
        ▼
ConditionalCallback：先求值 Conditional（条件，如属性 == X / count 达标 / median 越界）
        │  条件成立
        ▼
Callback 动作链（graph 索引到 generated.hpp 中的回调表）
   ├── elog<...>            创建 error log
   ├── journal              写 journal
   ├── MethodInvoke         调 D-Bus 方法
   └── EventManager.create  在 xyz.openbmc_project.Events 下建事件对象并持久化
```

main.cpp（`src/main.cpp`）流程很短：
1. `sdeventplus::Event::get_default()` 拿主事件循环；
2. 在 `OBJ_EVENT` 路径上挂 Object Manager；
3. `phosphor::events::getManager().restore()` —— 从持久化目录恢复上次的事件对象；
4. 遍历 `ConfigPropertyWatches` / `ConfigPathWatches`（generated.hpp 生成的表），
   先 `start()` 注册 D-Bus match，再各跑一次 `callback(Context::START)`
   （**启动上下文**：让开机时就对当前属性值做一次条件判断）；
5. 请求 bus name、进入 `event.loop()`。

注意 `Context` 有两个取值：`START`（开机全量检查）和 `SIGNAL`（信号触发），
写条件/回调时要意识到两者都会进来。

### 3.1 条件类深挖（count.hpp / median.hpp）

**CountCondition**（count.hpp:36）是最有代表性的聚合条件，求值是"两遍"语义：

1. 第一遍：对索引里**每个属性值**套用 `propertyOp`（如 `>= 阈值`），
   结果写回存储槽（storage[1]，filters.hpp 系列负责填充索引）并计数；
2. 第二遍：对计数结果套用 `countOp`（如 `>= N 个`）。
   即"**N 个传感器里至少 M 个越界**"这类规则，一条 YAML condition 就能表达。

`oneshot=true` 时条件触发一次后"锁存"，必须等条件回到 false 才允许再次触发
——避免同一批传感器持续越界时反复刷日志。

**MedianCondition**（median.hpp）同理但用中位数，适合"半数以上传感器共同漂移"
的场景。`CountCondition` 生成逻辑见 pdmgen.py 的 `CountCondition`（pdmgen.py:821）。

### 3.2 动作类深挖（elog.hpp / journal.hpp / method.hpp）

| 类 | 作用 | 细节 |
|---|---|---|
| `ElogBase` → `CallElog` / `Elog`（elog.hpp） | 记 phosphor-logging error log | 模板参数是生成的错误类型；`Elog` 在生成 C++ 端异常时报 `InternalFailure` |
| `ElogWithMetadataCapture`（elog.hpp:141） | 带元数据的 error log | 会回读 Storage 里的 pass/fail 结果，按属性顺序填元数据字段——count 条件 + 该动作 = "哪几个传感器坏了"自动可查 |
| `journal.hpp` | 直接写 systemd journal | `sendJournalRecord`，带 MESSAGE_ID |
| `method.hpp` | 调远端 D-Bus 方法 | 包装 `SDBusPlus::callMethod*`；可先经 mapper 反查服务名 |
| `format.hpp` | 格式化消息 | CaptureDef 从 Storage 抓字段拼字符串 |

### 3.3 事件对象持久化（event_manager.cpp）

`EventManager::create`（event_manager.cpp:31）：
- 事件按 eventName 分队列，对象路径 `OBJ_EVENT/<event名>/<id>`，
  id 取队尾路径文件名 +1；
- **容量封顶**：每类事件最多 `MAX_EVENTS` 条，满了删最老（连磁盘持久化文件一起删）；
- 事件以 sdbus 序列化文件存到 `EVENTS_PERSIST_PATH`（按 事件名/时间戳 组织），
  `restore()`（event_manager.cpp:88）开机递归扫描该目录重建对象并 `emit_object_added`。

即：pdm 里的"事件"是 phosphor-dbus-monitor 自有的一层轻量日志，
与 phosphor-logging 的 error log 并存，供界面/客户端按类别枚举。

## 4. YAML → 代码生成（改配置要懂的部分）

`pdmgen.py` 根据 YAML 里的段生成不同 C++ 对象，常见关键字段：

| YAML 段 | 生成内容 |
|---|---|
| `properties` | 属性组（路径/接口/属性名、meta/level），供条件与过滤引用 |
| `paths` / `pathgroups` | 监控的对象路径组 |
| `conditions` | 条件（property_state、count 达标、median 越界等），生成 IndexedConditional |
| `callbacks` | 动作（elog、journal、method call、事件创建、errorlog 报警），生成 Callback 表 |
| `propertywatches` / `pathwatches` | watch 声明：监听哪个接口/路径，绑定到哪个回调 |

生成规则在 `src/pdmgen.py`，各段 gtest 在 `src/test/*gentest*`。
`src/example/example.yaml` 是最好的阅读起点：先看它，再对照 `generated.hpp`（构建产物）理解映射。

## 5. 关键类速查

- **Watch**（`watch.hpp`）：接口 `start()` + `callback(Context)`。所有 watch 的基类。
- **SDBusPlus**（`sdbusplus.hpp`）：静态封装。`getBus()` 单例总线；`callMethod*` 系列发方法调用；
  `addMatch()` 把 match 注册进一个静态 vector 保活；`getBusName()` 用 mapper 反查服务名
  （OpenBMC 惯例：不写死服务名，用 mapper 的 GetObject 查）。
- **PropertyWatch / PropertyWatchOfType**（`propertywatch.hpp`）：
  模板参数决定"监听模式"—— `PropertyWatchOnSignals`（听 PropertiesChanged）
  或 `PropertyWatchOnInterfacesAdded`（听 InterfacesAdded）；
  `updateProperties()` 维护属性缓存（启动时 GetAll 一次 + 之后增量更新）。
- **Conditional / IndexedConditional**（`callback.hpp`）：条件求值，index 是属性缓存
  （`PropertyIndex = map<pair<path,interface>, map<property, Info>>`）。
- **ConditionalCallback**：把"条件 + 动作"绑在一起，通过 `graph` 索引找到真正的动作对象。
- **EventManager**（`event_manager.hpp`）：`create()` 在 D-Bus 上创建事件对象，
  `restore()` 开机重建。事件以 sdbus 序列化文件形式持久化（`event_serialize.*`）。
- **count.hpp / median.hpp**：高级条件 —— 同一属性在 N 个对象上满足条件 M 次、
  或中位数越过阈值才触发（用于多路传感器聚合告警）。

## 6. 构建与测试

- 构建系统：meson。在 OpenBMC 整机镜像里通常由 bitbake 打包为
  `phosphor-dbus-monitor`（YAML 由机器层 bbappend 提供）。
- 单独跑测试（需 meson 环境 + 依赖 sdbusplus/sdeventplus 等 phosphor 库）：

```bash
meson setup build && meson test -C build
```

- 快速理解生成逻辑：读 `src/example/*.yaml` + 对应 `src/test/*gentest.cpp` 的断言。

## 7. 新手学习路径（建议顺序）

1. `src/example/example.yaml` —— 看声明式配置长什么样；
2. `src/main.cpp` —— 60 行看懂启动流程；
3. `src/watch.hpp` → `src/propertywatch.hpp` → `src/callback.hpp` —— 三层抽象；
4. `src/pdmgen.py` 的 `emit*` 函数 —— YAML 字段到 C++ 表的映射；
5. `src/elog.hpp` / `src/event_manager.hpp` —— 动作侧的落地。

## 8. 常见改动场景

- **新增一条监控规则**：不改 C++，在机器层 YAML（bbappend 提供的 events/pdm 配置）
  里加 condition + callback + propertywatch，重新编译镜像即可。
- **新增回调类型**：在 `src/callback.hpp` 加类，在 `pdmgen.py` 加 YAML 关键字与 emit 逻辑，
  并补 `src/test/` 生成测试 —— 这是本仓库最典型的上游贡献模式。
- **调试**：运行时主要靠 journal 日志；用 `busctl` 观察
  `xyz.openbmc_project.Events` 下的事件对象与条件涉及的属性变化。
