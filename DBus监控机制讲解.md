# DBus 监控机制讲解：YAML 规则是怎么变成能跑的 C++ 的

> 面向 Java 背景、长期没碰 C++ 的你。主线只有一条：**"YAML 规则 → 生成 C++ → 运行时链路"**。
> 所有例子都来自本地 `phosphor-dbus-monitor` 仓库，带 `文件:行号`，可跳转核对。
> 📌 路径说明：源码在 `phosphor-dbus-monitor/src/` 下，正文引用形如 `src/count.hpp:36`；
> mako 模板在 `src/templates/`；构建配置在仓库根 `meson.build` 和 `src/meson.build`；
> 示例配置在 `src/example/`。

---

## 0. 先讲主线：这条规则经历了什么

业务文档里那条"4 个传感器里至少 3 个 >= 115 度就记 error log"的规则
（`src/example/example.yaml:207-238`），从文本到生效，会经历**两个截然不同的阶段**：

```
【构建期】YAML ──pdmgen.py──▶ generated.hpp（一堆 C++ 数组 + 对象）  ← 只做一次
【运行期】generated.hpp 里的对象 ──start()/callback()──▶ 监听→求值→动作  ← 一直在跑
```

把这条主线拆成 5 个关键问题，本文逐个回答：

1. YAML 里的 condition/callback/watch 是怎么被解析、被引用的？（第 2、3 节）
2. 解析完怎么"渲染"成 C++？生成物长什么样？（第 3、4 节）
3. 运行时属性变化是怎么一步步传到动作的？（第 5 节总图 + 第 6~10 节拆解）
4. `CountCondition` 为什么要"两遍求值"？（第 8 节）
5. `oneshot` 是怎么"锁存"防刷屏的？（第 9 节）

> 一句话先剧透：**pdmgen.py 把 YAML 里的"名字引用"全部解析成"数组下标"，渲染进 generated.hpp 的静态数组；
> 运行时 main() 把这些数组里的对象逐个 start()（挂 match）、逐个 callback()（求值触发），
> 之后靠 D-Bus 信号驱动的缓存更新 + 条件求值 + callback 索引完成整条链路。**

---

## 1. 先看一条真实的 YAML 规则长什么样

机制要讲清楚，得先有"输入"。看示例里的 count condition（`src/example/example.yaml:229-238`）：

```yaml
- name: example count condition          # 规则名（引用靠它）
  class: condition                       # 这是一个"条件"
  condition: count                       # 条件类型：count（另一种见 median）
  paths: example path group             # 引用一个 path group（4 个对象路径）
  properties: example property group    # 引用一个 property group（ValueA/ValueB）
  callback: example callback group      # 条件成立后要触发的 callback（组）
  countop: ">="                          # 第二遍：数量 >= ...
  countbound: 3                          # ...3 个
  op: ">="                               # 第一遍：每个属性值 >= ...
  bound: 115                             # ...115
  oneshot: true                          # 触发一次后锁存，直到条件变 false 才解锁
```

配套的 watch（`src/example/example.yaml:63-73`）告诉框架"盯哪些对象的哪些属性、条件成立后调谁"：

```yaml
- name: example property watch
  class: watch
  watch: property
  paths: example path group             # 监听这些路径
  properties: example property group    # 上的这些属性
  callback: example count condition     # ★属性变化时先跑这个条件
  filters:                              # 值必须落在 [0,100]... 咦，115 会被过滤？
      - {op: ">=", bound: 0}
      - {op: "<=", bound: 100}
  ignore_start_callback: true
```

（示例里 filter 上界 100 只是演示过滤用法，和 115 阈值矛盾是示例的随意之处——
真实配置里二者要协调。这正好提醒你：**filter 会把不符合的值从缓存清掉**，`src/filters.hpp:55-77`。）

**Java 类比**：这段 YAML 就是你的"规则配置中心"（类似一个 Spring 的 `@Configuration` + 表达式），
`name` 是 Bean 名，`paths/properties/callback` 是对其他 Bean 的**按名引用**——
接下来 pdmgen.py 干的事，就是把这些"按名引用"解析成"数组下标"（类似 Spring 容器帮你按名字注入 Bean，
只是这里在**构建期**就完成了）。

---

## 2. 构建期：pdmgen.py 怎么读 YAML

### 2.1 谁在什么时候调用它

生成发生在 meson 构建里。`src/meson.build:24-44` 定义了一个 `custom_target('generated.hpp')`，
真正执行的命令是（节选 `src/meson.build:26-40`）：

```
python3 src/pdmgen.py
    -t generated.hpp.mako        # 用哪个模板
    -p src/templates             # 模板搜索目录
    -d <YAML_PATH>               # 输入 YAML 目录
    -o <build>/generated.hpp     # 输出
    -e <build>/errors.hpp        # 错误类型头文件
    generate-cpp                 # 子命令
```

`YAML_PATH` 没配置时默认指向 `src/example`（`src/meson.build:19-22`）；
在整机镜像里由机器层 bbappend 提供真实 YAML。命令行解析在 `src/pdmgen.py:1304-1374`，
其中模板默认值 `generated.mako.hpp` 定义在 `src/pdmgen.py:1327`，子命令注册表在 `src/pdmgen.py:1306-1308`。

### 2.2 四步工作流

`src/pdmgen.py:3-16` 的模块注释把流程写得很清楚，翻译成人话：

```
① Import YAML：把每个 .yaml 读成 Python 原生 dict/list
② Everything.load：把所有 YAML 聚合成一个 Everything 对象
③ Everything 构造函数：把原生数据转成"渲染辅助类"（Renderer 子类）
④ generate_cpp：用 mako 模板把辅助类渲染成 C++，写到 generated.hpp
```

对应代码：

**第一步 + 聚合**在 `Everything.load`（`src/pdmgen.py:1218-1244`）：

```python
def load(args):
    objs = {}
    yaml_files = filter(lambda x: x.endswith(".yaml"),
                        os.listdir(args.inputdir))              # :1224 扫描所有 yaml
    for x in sorted(yaml_files):                                # :1226 排序，保证确定性
        with open(path, "r") as fd:
            Everything.load_one_yaml(path, fd, objs)            # :1229 解析进 objs
    # ★关键：所有对象建好后，把"按名引用"解析成"数组下标"
    for cls, items in objs.items():
        for obj in items:
            obj.setup(objs)                                     # :1237-1240
    return Everything(**objs)                                   # :1242
```

**第二步 + 第三步**的关键就是那个 `setup(objs)` 循环。注释（`src/pdmgen.py:1232-1236`）说得很直白：
"配置项之间通过 name 互相引用，但渲染时引用只是一个数组下标；现在把所有引用解析成下标。"
例如 `ConditionCallback.setup`（`src/pdmgen.py:778-786`）把条件名解析成 condition 数组下标：

```python
self.graph = get_index(objs, "callbackgroup", [self.instance], ...)   # :779 回调组下标
self.condition = get_index(objs, "condition", self.name, ...)         # :783 条件下标
```

`get_index`（`src/pdmgen.py:52-65`）就是"按名字找下标"的工具函数。
这一步类比 Java：相当于 Spring 容器在启动时解析 `@Autowired` 的 by-name 依赖，
只不过 PDM 把它提前到了构建期。

**第四步** 在 `Everything.generate_cpp`（`src/pdmgen.py:1269-1282`）：

```python
def generate_cpp(self, loader):
    open(args.gen_errors, "w+")                 # :1272 先清空 errors.hpp
    with open(args.output, "w") as fd:
        fd.write(self.render(loader, args.template, ...))  # :1274-1282 渲染模板
```

它把所有解析好的数组（meta/properties/interfaces/watches/callbacks/conditions…）作为参数
传给 mako 模板 `generated.hpp.mako`。

### 2.3 几个需要记住的生成类

pdmgen.py 里每个 YAML 指令都有一个对应的 `Renderer` 类。重点看三个，后面讲运行时会用到：

| YAML 指令 | 生成类 | 关键行为 | 位置 |
|---|---|---|---|
| `condition: count` | `CountCondition` | 读出 `countop/countbound/op/bound/oneshot`，渲染 `count.cpp.mako` | `src/pdmgen.py:825-846` |
| `condition`（通用） | `Condition` | `factory()` 自动为条件创建一个 `ConditionCallback` 绑定 callback | `src/pdmgen.py:797-823` |
| `watch: property` | `PropertyWatch` | `setup()` 把 callback / filters 解析成下标 | `src/pdmgen.py:645-718` |
| `condition + defer` | `ConditionCallback` | 有 `defer` 就渲染成 DeferrableCallback，否则普通 ConditionalCallback | `src/pdmgen.py:753-797` |

> 📌 **工厂模式**：`Condition.factory`（`src/pdmgen.py:805-823`）会自动"变出"一个 `ConditionCallback`，
> 把你写的条件和要触发的回调绑在一起。所以你在 YAML 里只写了 condition，
> 生成物里却多出一个 `ConditionalCallback` 对象——这就是工厂在背后帮你做的。
> 类比 Java：`@Bean` 方法里 `new` 出关联对象并注入依赖。

---

## 3. 生成物结构：generated.hpp 里到底有什么

`generated.hpp.mako`（`src/templates/generated.hpp.mako`）渲染后，产出一组 `Config*` 结构，
每个都提供一个静态 `get()` 返回一张**静态表**（`std::array` / `std::vector`）。
这就是 yaml → C++ 的核心映射。看几个关键的：

| 生成的结构 | 里面装什么 | 模板位置 |
|---|---|---|
| `ConfigPaths` / `ConfigInterfaces` / `ConfigProperties` | 路径、接口、属性名的字符串表 | `src/templates/generated.hpp.mako:50-80, 98-112` |
| `ConfigPropertyStorage` | 每个属性的存储槽：`array<tuple<any, any>, N>`，一个存值、一个存结果 | `src/templates/generated.hpp.mako:114-123` |
| `ConfigPropertyIndicies` | 属性索引表（`PropertyIndex`），把"路径+接口+属性→值/结果槽"编好号 | `src/templates/generated.hpp.mako:149-182` |
| `ConfigConditions` | 所有条件的数组（`array<unique_ptr<Conditional>, N>`） | `src/templates/generated.hpp.mako:224-238` |
| `ConfigPropertyCallbacks` | 所有动作的数组（`array<unique_ptr<Callback>, N>`） | `src/templates/generated.hpp.mako:240-254` |
| `ConfigPropertyCallbackGroups` | 回调组（graph）：一组动作在回调数组里的下标 | `src/templates/generated.hpp.mako:207-222` |
| `ConfigPropertyWatches` | 所有 watch 的数组，每个是 `PropertyWatchOfType<类型, SDBusPlus>` | `src/templates/generated.hpp.mako:273-307` |
| `ConfigPathWatches` | 路径 watch 数组（`PathWatch<SDBusPlus>`） | `src/templates/generated.hpp.mako:309-332` |

### 3.1 属性索引 PropertyIndex：整条链路的"数据底座"

`PropertyIndex` 定义在 `src/data_types.hpp:87-90`，本质是个嵌套 map：

```
PropertyIndex :
   key   = (path, interface, property)         ← 三元组定位一个属性
   value = (pathMeta, propertyMeta, storage)   ← 元数据 + 一个存储槽引用
                                     ↑
                            storage = (value, result)  ← 值槽 + 结果槽
```

三个位置常量在 `src/data_types.hpp:25-36` 定义得清清楚楚：

```
pathIndex=0, interfaceIndex=1, propertyIndex=2     // key 的三个字段
pathMetaIndex=0, propertyMetaIndex=1, storageIndex=2 // value 的三个字段
valueIndex=0, resultIndex=1                        // storage 的两个槽
```

**为什么要有 result 槽？** 因为 `CountCondition` 两遍求值时，第一遍把每个属性"是否达标"写进 result 槽
（`src/count.hpp:72`），第二遍统计达标数量；后面 `ElogWithMetadataCapture` 又从 result 槽里
读出"哪些属性达标"来拼日志明细（`src/elog.hpp:189-201`）。一个 result 槽，串起了条件判定和日志元数据。

**Java 类比**：`PropertyIndex` 相当于一个 `Map<PropertyKey, PropertySlot>`，
`PropertySlot` 里有两个可变字段 `value` 和 `result`——
全是共享引用（`TupleOfRefs`），所以 watch 改 value、condition 读 value/写 result、callback 读 result，
操作的都是同一块内存，不需要拷贝。

### 3.2 生成物里的对象是怎么 new 出来的

看 `ConfigConditions` 的模板（`src/templates/generated.hpp.mako:224-238`）：

```cpp
using Conditions = std::array<std::unique_ptr<Conditional>, ${len(conditions)}>;
static auto& get() {
    static const Conditions propertyConditions = {
        ${c.construct(loader, indent=indent +3)},   // :233 每个条件 construct 出来
        ...
    };
    return propertyConditions;
}
```

`c.construct()` 会调用对应子模板。比如 count 条件渲染成 `src/templates/count.cpp.mako:1-5`：

```cpp
std::make_unique<CountCondition<类型>>(
    ConfigPropertyIndicies::get()[下标],                    // 属性索引
    [](const auto& item){return item >= 3;},                // countOp（第二遍）
    [](const auto& item){return item >= 115;},              // propertyOp（第一遍）
    true)                                                   // oneshot
```

**这里的两个 Lambda 就是第 8 节要讲的两遍求值**——YAML 的 `countop/countbound` 和 `op/bound`
被原样编译成了两个 `std::function`。
类比 Java：这就是把配置里的比较符"代码生成"成了两个 `Predicate<T>` 和 `Predicate<Integer>`。

`ConditionalCallback` 由 `src/templates/conditional.cpp.mako:1-10` 渲染：有 `defer` 就 `make_unique<DeferrableCallback>`，
否则 `make_unique<ConditionalCallback>`，构造参数是 graph 下标 + 条件引用。

---

## 4. 运行时链路总图（先把骨架看清）

下面这张 ASCII 图是全文的骨架。先记住它，后面逐段拆：

```
                         【构建期：已完成】
   YAML (example.yaml) ──pdmgen.py──▶ generated.hpp
        conditions/callbacks/watches ─▶ Config* 静态表 + Lambda


                         【运行期：main() 启动】
   main() (src/main.cpp:28)
     ├─ getBus / 挂 ObjectManager / restore()          :30-36
     ├─ 对每个 PropertyWatch: start()                   :38-41  ┐
     ├─ 对每个 PropertyWatch: callback(START)           :43-46  │ 开机全量检查
     ├─ 对每个 PathWatch: start()/callback(START)       :47-55  ┘
     └─ request_name + event.loop()                     :57-59  进入事件循环
              │
              │ D-Bus 信号来了（PropertiesChanged / InterfacesAdded）
              ▼
   ┌──────────────────────────────────────────────────────────┐
   │ PropertyWatch::start() 注册的 match 回调                 │
   │   InterfacesAdded  → interfacesAdded()  (propertywatchimpl.hpp:44-46) │
   │   PropertiesChanged→ propertiesChanged() (propertywatchimpl.hpp:72-79)│
   └──────────────────────────────────────────────────────────┘
              │  unpack 消息，定位 (path, interface, property)
              ▼
   PropertyWatchOfType::propertiesChanged()   (propertywatchimpl.hpp:143-181)
     ├─ 跑 filters（不符 → 清掉 value 槽）        :158-172
     ├─ 写 value 槽                              :176
     └─ callback(Context::SIGNAL)                :178
              │
              ▼
   PropertyWatch::callback()  → (*cb)(ctx)       (propertywatchimpl.hpp:115-128)
              │  cb 就是生成物里的 ConditionalCallback
              ▼
   ConditionalCallback::operator()(ctx)          (callback.hpp:178-184)
     ├─ if (condition())  ← ★求值（见下）
     │      └─ (*CallbackAccess::get()[graph[0]])(ctx)   :182  按 graph 下标取动作
              │
              ▼
   CountCondition::operator()() 两遍求值          (count.hpp:54-90)
     第一遍 count_if: propertyOp(value) → 写 result 槽  :58-75
     第二遍 countOp(count) → result                   :78
     oneshot 锁存判断                                  :83-89
              │ condition() 为 true
              ▼
   动作链 Callback（按 graph 下标继续）：
     ├─ GroupOfCallbacks::operator()  遍历 graph  (callback.hpp:147-153)
     ├─ Elog / ElogWithMetadataCapture  记日志     (elog.hpp)
     ├─ journal                                    (journal.hpp)
     ├─ MethodInvoke                                (method.hpp)
     └─ EventManager::create  建事件对象+持久化    (event_manager.cpp:31-86)
```

**读图要点**：整条链路里没有一行代码在解析 YAML；所有"配置"都变成了
`generated.hpp` 里的静态数组和 Lambda；运行时的"规则匹配"就是普通的虚函数调用 + 数组下标访问。
这在 Java 里相当于：配置在构建期编译成了 Bean 定义类，运行时只是按 Bean 名（这里变成下标）取对象调用。

---

## 5. 启动阶段：start() 和 callback(START) 为什么要分两个循环

`main()`（`src/main.cpp:28-59`）里有个容易忽略的细节：它把 watch 遍历了两遍——

```cpp
for (auto& watch : ConfigPropertyWatches::get()) { watch->start(); }        // :38-41
for (auto& watch : ConfigPropertyWatches::get()) { watch->callback(Context::START); } // :43-46
```

**为什么要分两个循环？** 因为第二遍的"开机全量求值"必须等**所有** watch 都 `start()` 完、
把所有属性的当前值都拉进缓存之后再做，否则某个条件引用了还没初始化的属性，就会误判。

`start()` 里做了三件事（`src/propertywatchimpl.hpp:27-112`）：

1. **`alreadyRan` 防重入**（`:29-32`）：`start()` 只允许成功执行一次。
2. **`convert(index)` 重组索引**（`:36`）：把扁平的 `PropertyIndex` 按 `path → interface → property`
   重排成嵌套 map（`src/propertywatch.cpp:33-46`），方便按 mapper 响应的结构查。
3. **对每个 path 注册两类 match + 拉一次缓存**（`:38-109`）：
   - `InterfacesAdded` match（`:44-46`）——监听"对象/接口出现"；
   - mapper `GetObject` 查询服务名（`:50-65`）——注意这里体现了 OpenBMC 惯例：
     **不写死服务名，用 mapper 反查**（`src/sdbusplus.hpp:97-120` 的 `getBusName` 同理）；
   - `PropertiesChanged` match（`:72-79`）——监听属性变化；
   - `updateProperties()` 调 `GetAll` 拉满缓存（`:99`，实现在 `:131-140`）。

**Java 类比**：这就是两阶段初始化——`start()` 相当于 `@PostConstruct`（建立订阅、预热缓存），
`callback(START)` 相当于应用就绪后主动触发的一次"全量对账"。

---

## 6. match 是怎么挂上去的：addMatch 与保活

无论是属性还是接口 watch，最终都调到 `SDBusPlus::addMatch`（`src/sdbusplus.hpp:90-94`）：

```cpp
static auto addMatch(const std::string& match,
                     sdbusplus::match::callback_t&& callback)
{
    getWatches().emplace_back(getBus(), match, std::move(callback));  // :93
}
```

关键在 `getWatches()`（`src/sdbusplus.hpp:27-31`）：它是一个**函数内的 static vector**。

**为什么要这个 vector？** sdbusplus 的 `match` 对象在析构时会自动**取消订阅**。
如果注册完就把临时对象扔掉，match 立刻失效。放进 static vector 相当于"持有强引用保活"——
这和 Java 里把 listener 加进一个静态 `List` 防止被 GC 是**同一个套路**。

**lambda 捕获 `this`**：注册时传的是 `[this](auto& msg){ ... }`（`src/propertywatchimpl.hpp:74-79`）。
当信号到来，sdbusplus 调用这个 lambda，进而调回对应的 watch 对象。

---

## 7. 属性变化 → 更新缓存 → 触发求值

信号到来后走 `PropertyWatchOfType::propertiesChanged`（`src/propertywatchimpl.hpp:143-181`）：

```cpp
void propertiesChanged(path, interface, properties) {
    for (const auto& p : properties) {                     // :148 遍历变化了的属性
        auto key = std::make_tuple(path, interface, p.first);
        auto item = this->index.find(key);                 // :151 只关心被监控的属性
        if (item == this->index.end()) continue;           // :152-156 不关心的忽略
        // 跑过滤器：任一过滤不通过 → 清掉 value 槽
        if (filterOps && (*filterOps)(anyValue)) {         // :162-172
            isFiltered = true;
            std::get<valueIndex>(storage.get()).reset();   // :170 清掉
        }
        if (!isFiltered) {
            std::get<valueIndex>(storage.get()) = value;   // :176 ★写 value 槽
            this->callback(Context::SIGNAL);               // :178 ★触发求值
        }
    }
}
```

三个要点：

1. **只处理被监控的属性**（`:151-156`）：总线上的信号再多，不在 `PropertyIndex` 里的直接跳过。
2. **过滤器决定属性是否"存在"**（`:158-172`）：被过滤掉的属性会把 value 槽 `reset()`，
   相当于"当作没这个属性"——`CountCondition` 第一遍遇到 `has_value()==false` 的不计数
   （`src/count.hpp:64-67`）。这就是 filter 影响条件判定的机制。
3. **写完值立刻触发**（`:176-178`）：每个属性变化都触发一次求值（注意是 SIGNAL 上下文）。

`callback()` 本体（`src/propertywatchimpl.hpp:115-128`）只是做两件小事：
START 且设了 `ignoreStartCallback` 就直接返回（`:119-122`）；否则转发给绑定的 `cb`（`:124-127`）。
这个 `cb` 就是 `generated.hpp` 里构造 watch 时传进去的 `*ConfigPropertyCallbacks::get()[下标]`
（`src/templates/generated.hpp.mako:293-295`）——也就是那个 `ConditionalCallback`。

**Java 类比**：`PropertyWatch` 是 Subject（被观察者），`ConditionalCallback` 是 Observer。
`callback()` 就是 `subject.notifyObservers(event)` 那一句。

---

## 8. 核心：CountCondition 为什么"两遍求值"

这是全文最关键的一节。`CountCondition`（`src/count.hpp:36`）的 `operator()`（`src/count.hpp:54-90`）做了两遍扫描：

### 8.1 第一遍：对每个属性值套用 propertyOp

```cpp
auto count = std::count_if(index.cbegin(), index.cend(), [this](const auto& item) {
    const auto& storage = std::get<storageIndex>(item.second);          // :62
    if (!std::get<valueIndex>(storage.get()).has_value()) return false; // :64 不存在的不计
    const auto& value = std::any_cast<T>(std::get<valueIndex>(storage.get())); // :68-69
    auto r = propertyOp(value);                                          // :70 ★属性值 >= 115 ?
    std::get<resultIndex>(storage.get()) = r;                            // :72 ★写 result 槽
    return r;
});
```

这一遍把 YAML 的 `op: ">=" bound: 115` 编译成的 Lambda（`src/templates/count.cpp.mako:4`）
作用到每个属性值上，得到"这个传感器是否越界"，**顺手把 true/false 写进 result 槽**。

### 8.2 第二遍：对"计数结果"套用 countOp

```cpp
auto result = countOp(count);   // :78 ★"达标的个数 >= 3 ?"
```

这一遍用 YAML 的 `countop: ">=" countbound: 3` 编译成的 Lambda（`src/templates/count.cpp.mako:3`）
判断"达标数量够不够"。

### 8.3 为什么非要两遍？（这是设计精髓）

因为要表达"**N 个对象里至少 M 个满足某条件**"这种**跨对象的聚合规则**，
单次比较做不到：你得先知道"每个对象各自的判定结果"，再对这些结果做统计。

```
单遍只能表达：单个属性值 >= 115       （propertyOp 一项）
两遍才能表达：4 个传感器里 >=3 个满足 （propertyOp + countOp 组合）
```

**Java 类比**：这就像 Stream API 的两段式操作——
`list.stream().map(propertyOp).filter(Boolean::booleanValue).count()` 再
`countOp(count)`，即"先 map 再 reduce"；`propertyOp` 是 `Predicate<T>`，`countOp` 是 `Predicate<Long>`。
一个 YAML condition 声明就把这整条流定义了。

### 8.4 result 槽的"一鱼两吃"

第一遍写入 result 槽不只是为了第二遍计数服务，它还有第二个用途：
`ElogWithMetadataCapture::captureMetadata`（`src/elog.hpp:184-206`）会遍历索引，
把 result 槽为 true 的属性拼成 `|path:name=value|...` 的元数据字符串（`src/elog.hpp:189-201`）。
于是"哪几个传感器超限"的明细自动生成——这就是业务文档里说的"日志里自动带上超限传感器"。

**两个成员变量的角色**（`src/count.hpp:92-101`）：

```
countOp    : std::function<bool(size_t)>  第二遍的"数量比较器"
propertyOp : std::function<bool(T)>        第一遍的"属性值比较器"
oneshot    : const bool                    是否锁存
lastResult : bool                          上次求值结果（锁存用）
```

> 📌 顺带一提：`MedianCondition`（`src/pdmgen.py:848-868`）是同样的两遍思想，只是把"计数"
> 换成"求中位数"，适合"一组传感器整体漂移"的场景。

---

## 9. oneshot 锁存：为什么不会刷屏

`CountCondition` 的第三件事是锁存判断（`src/count.hpp:80-89`）：

```cpp
auto result = countOp(count);                 // :78 本轮逻辑结果
if (oneshot && result && lastResult) {        // :83 锁存条件：开了一枪 + 上轮也成立
    return false;                             // :85 本轮不再放行
}
lastResult = result;                          // :88 记住本轮结果
return result;                                // :89
```

逐项理解：

| 情形 | `oneshot && result && lastResult` | 返回值 | 效果 |
|---|---|---|---|
| 条件首次成立（lastResult=false） | false | true | ★触发第一次动作 |
| 持续越界、条件连续成立（lastResult=true） | true | false | 锁存，不重复触发 |
| 条件变 false（result=false，lastResult 更新为 false） | false | false | 不触发，并"解锁" |
| 条件再次成立（上次 false） | false | true | ★重新允许触发 |

翻译成人话：**"连续为真只算一次，中间必须至少假一次才能再次触发"**。
这就是业务文档说的"防止同一批传感器持续越界时反复刷日志"。

**Java 类比**：这像一个"边沿触发器"（edge-triggered latch）——只关心条件从 false→true 的**上升沿**，
忽略持续为 true 的电平。也可以理解为带状态的装饰器：`lastResult` 是这个装饰器的内部状态。

> 📌 配套细节：`ignore_start_callback`（`src/propertywatchimpl.hpp:119-122`）和
> `ElogWithMetadataCapture` 在 START 时直接 return（`src/elog.hpp:158-162`），
> 都是为了避免"开机那一下"产生多余动作——它们和 oneshot 一起构成完整的"防误报/防刷屏"策略。

---

## 10. 条件成立后：动作链怎么执行

`ConditionalCallback::operator()`（`src/callback.hpp:178-184`）条件成立后，按 graph 下标取动作：

```cpp
if (condition()) {
    (*CallbackAccess::get()[graph[0]])(ctx);   // :182 graph[0] 是"回调组"的下标
}
```

这里 `CallbackAccess` 就是生成的 `ConfigPropertyCallbacks` 表（`src/templates/generated.hpp.mako:240-254`）。
如果 callback 是一个**组**（多条动作），会走 `GroupOfCallbacks::operator()`（`src/callback.hpp:147-153`）：

```cpp
for (auto e : graph) { (*CallbackAccess::get()[e])(ctx); }  // :149-152 逐个执行
```

`graph` 是动作在回调数组里的下标集合——由 `src/templates/generated.hpp.mako:207-222` 的
`ConfigPropertyCallbackGroups` 生成。动作类型包括：

| 动作 | 作用 | 代码 |
|---|---|---|
| `Elog` / `CallElog` | 通过 `phosphor::logging::report<T>` 记 error log | `src/elog.hpp:67-93`（`report` 在 `:78`） |
| `ElogWithMetadataCapture` | 记日志并从 result 槽抓达标属性拼明细 | `src/elog.hpp:141-206` |
| `journal` | 写 systemd journal（带 MESSAGE_ID） | `src/journal.hpp` |
| `method` | 调远端 D-Bus 方法，必要时 mapper 反查服务名 | `src/method.hpp`、`src/sdbusplus.hpp:42-87` |
| `EventManager::create` | 在 `OBJ_EVENT` 下建事件对象并序列化持久化 | `src/event_manager.cpp:31-86` |

> 📌 **延迟动作**：如果 YAML 配了 `defer`，`ConditionalCallback` 会被换成 `DeferrableCallback`
> （`src/pdmgen.py:753-797`、`src/templates/conditional.cpp.mako:1-5`）。
> 它的 `operator()`（`src/callback.hpp:232-260`）在条件首次成立时启动一个定时器，
> 到期且条件仍成立才真正触发；中途条件变 false 就取消定时器（`:255-259`）。
> 这就是"抖动信号要稳定一段时间才算数"的实现——很像 Java 的 `debounce`。

---

## 11. 事件对象的持久化（动作之后的"记账"）

`EventManager::create`（`src/event_manager.cpp:31-86`）值得单独一看，因为它体现了"轻量台账"的完整实现：

- **id 生成**（`:53-59`）：取该 eventName 队列最后一个对象的路径文件名 +1；
- **对象路径**（`:65`）：`OBJ_EVENT/<eventName>/<id>`，`OBJ_EVENT` 定义在根 `meson.build:15`；
- **容量封顶**（`:70-78`）：达到 `MAX_EVENTS`（根 `meson.build:23`，值 20）就删最老的对象连同磁盘文件；
- **序列化**（`:84`）：写盘到 `EVENTS_PERSIST_PATH`（根 `meson.build:17-20`，`/var/lib/phosphor-dbus-monitor/events`）。

开机时 `main()` 调 `getManager().restore()`（`src/main.cpp:36`），
它递归扫描持久化目录（`src/event_manager.cpp:88-127`），反序列化并 `emit_object_added()` 重建对象
（`src/event_manager.cpp:115-118`）。

**Java 类比**：这就是一个带 LRU 上限的持久化事件队列，`restore()` 相当于应用启动时的"从磁盘回放重建内存状态"。

---

## 12. 排错速查表（机制类）

| 现象 / 疑问 | 机制层面的正解 |
|---|---|
| "改了 YAML，行为没变？" | YAML 是构建期生成的，必须重新跑 `pdmgen.py`/重新构建；运行时不读 YAML（`src/meson.build:24-44`） |
| "怎么确认 YAML 被正确解析了？" | 看构建产物 `generated.hpp` 里的 `Config*` 数组下标对不对；生成逻辑 `src/pdmgen.py:1269-1282` |
| "为什么生成物里条件比 YAML 多一个对象？" | `Condition.factory()` 自动创建了 `ConditionCallback`（`src/pdmgen.py:805-823`） |
| "condition 里的 graph 下标是什么意思？" | 指向 `ConfigPropertyCallbacks`/`ConfigPropertyCallbackGroups` 的下标，按名引用在 `setup()` 里解析成下标（`src/pdmgen.py:778-786`） |
| "属性变化了，回调没触发？" | 查 filter 是否把它清掉了（`src/propertywatchimpl.hpp:162-172`）；或 match 没注册上（`src/propertywatchimpl.hpp:72-79`） |
| "开机时超温没告警？" | `ignore_start_callback: true` 跳过了 START（`src/propertywatchimpl.hpp:119-122`） |
| "条件一直为真却只触发一次？" | 这是 `oneshot` 锁存，条件必须变 false 一次才重新解锁（`src/count.hpp:83-89`） |
| "result 槽是干嘛的？" | 存每个属性第一遍求值的 true/false，供第二遍计数和 elog 元数据使用（`src/count.hpp:72`、`src/elog.hpp:189-201`） |
| "match 会不会注册完就失效？" | 不会，`addMatch` 把 match 放进 static vector 保活（`src/sdbusplus.hpp:27-31, 90-94`） |
| "服务名写死在哪里？" | 不写死，用 mapper `GetObject` 反查（`src/sdbusplus.hpp:97-120`、`src/propertywatchimpl.hpp:50-65`） |
| "想加一种新回调类型？" | 在 `src/callback.hpp` 加类，在 `src/pdmgen.py` 加 YAML 关键字+emit，在 `src/templates/` 加 mako，补 `src/test/*gentest.cpp` |

---

## 13. Java 类比速查表

| PDM 机制 | Java 类比 | 说明 |
|---|---|---|
| YAML 配置 | Spring `@Configuration` / 规则配置中心 | 声明"条件 + 动作" |
| pdmgen.py | 构建期代码生成器（如 MapStruct / annotation processor） | 把配置编译成代码 |
| `name` 引用 → 数组下标 | Spring 按名注入 `@Autowired` | `setup()` 在构建期解析引用 |
| `Config*` 静态表 | 单例 Bean 注册表 / `Map<String, Object>` | 运行时按表取对象 |
| `PropertyIndex` | `Map<Key, Slot>`，Slot 含 value/result | 共享引用的属性缓存底座 |
| `Watch` / `PropertyWatch` | 观察者模式的 Subject | `start()`≈`@PostConstruct`，`callback()`≈`notifyObservers` |
| `addMatch` 的 static vector | 静态 `List<Listener>` 保活，防 GC | 防止订阅被取消 |
| `ConditionalCallback` | 带 `if` 的 Decorator / `Predicate` + 动作 | 条件通过才调用动作 |
| `propertyOp` / `countOp` 两个 Lambda | `Predicate<T>` + `Predicate<Long>`（map→reduce） | 两遍求值 |
| `oneshot` / `lastResult` | 边沿触发器（edge-triggered latch） | 连续为真只算一次 |
| `DeferrableCallback` | `debounce` / `ScheduledExecutorService` 延迟任务 | 稳定一段时间才触发 |
| `unique_ptr<Callback>` | 持有对象引用的容器元素 | 生成物用智能指针管理对象生命周期 |

---

## 14. 一句话总结

> **pdmgen.py 在构建期把 YAML 里"按名字互相引用"的 conditions/callbacks/watches 解析成数组下标，
> 连同比较符编译成的 Lambda 一起渲染进 `generated.hpp` 的 `Config*` 静态表；
> 运行时 `main()` 把 watch 逐个 `start()`（挂 D-Bus match、GetAll 预热缓存）再逐个 `callback(START)` 全量检查，
> 之后信号到来 → 更新 value 槽 → `callback(SIGNAL)` → `ConditionalCallback` 先跑 `CountCondition` 两遍求值
> （propertyOp 逐项判定写 result 槽、countOp 统计数量，oneshot 靠 lastResult 锁存防刷屏）
> → 条件成立按 graph 下标调用动作链（elog/journal/method/事件对象）。
> 全程不解析 YAML——规则早已"编译"进二进制，这就是它高效、松耦合、配置驱动的机制精髓。**

> 配套：业务背景看《DBus监控业务详解.md》，代码结构看 `ONBOARDING.zh-CN.md`，
> 流程可视化看 `业务流程交互图.svg`。
