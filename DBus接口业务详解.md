# DBus接口业务详解：一句"开机"背后，全 BMC 都在查的同一本字典

> **写给谁**：有 openUBMC 经历、7 年没碰 C++ 的 Java 程序员，正在系统学 OpenBMC。
> **本文回答什么问题**：这个叫 `phosphor-dbus-interfaces` 的仓库里没有一个提供业务的后台进程，为什么它却是 BMC 上"开机""记日志""读传感器"这些事能被不同进程互相听懂的起点？
> **讲法**：先把真实代码和真实调用链摆出来，再点概念；C++ 的东西尽量用 Java 的 `interface`、注解处理器、protobuf 生成代码去类比。
> **路径说明**：下文所有 `path:行号` 都相对本仓库根目录 `phosphor-dbus-interfaces/`，行号可直接跳转核对。不上环境搭建，只看代码怎么组织的。

---

## 0. 一句话说清业务

如果你在 openUBMC 里写过"规范/接口定义"，这里的概念可以直接搬过来：**这个仓库是 OpenBMC 全体服务共用的一本 D-Bus 接口字典，它自己不跑任何服务，只负责把接口长什么样写清楚。**

README 第一句就说得很直白：

```text
YAML descriptors of standard D-Bus interfaces.
The format is described by the sdbusplus binding generation tool sdbus++.
                                        —— README.md:3-4
```

逐字拆一下这句：

- `YAML descriptors`：这里放的是描述文件，不是业务代码。全仓库 437 个 `.yaml`，其中 **347 个是 `*.interface.yaml`**（接口定义本体），另外是 `39` 个 `*.errors.yaml`、`24` 个 `*.events.yaml`、`27` 个 `*.metadata.yaml`。
- `standard D-Bus interfaces`：这些描述是"标准"，也就是大家约定好都这么用。
- `sdbusplus binding generation tool sdbus++`：格式由外部工具定义，这个工具不在本仓库里，通过 `subprojects/sdbusplus.wrap:1-6` 拉进来（`program_names = sdbus++, sdbus++-gen-meson`，`subprojects/sdbusplus.wrap:6`）。

注意一个容易误解的点：说它"不跑进程"不等于"什么都不产出"。它会**产出一个共享库和一堆头文件**：

```meson
libphosphor_dbus = library(
    'phosphor_dbus',
    generated_sources,
    ...
    install: true,
)
                                        —— meson.build:77-85
```

但它不会像真正的服务那样占住一个 D-Bus service name 去等人调用。它甚至会在 YAML 里写明"这个接口默认谁提供"（比如 `service_names: default: xyz.openbmc_project.Logging`，`yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:123-124`），但那只是写给实现方的说明，本仓库自己并不启动 `xyz.openbmc_project.Logging`。

> **Java 类比**：把它想成一个只有 `interface` / `enum` / 注解定义、没有 `main` 方法的公共模块。你的业务服务 `implements` 这些接口，客户端拿生成的 proxy 去调。本仓库就是那个 `.java` 源文件集合；`server.hpp` / `client.hpp` 是"编译时生成的桩代码"，相当于 protobuf 生成的 `XxxGrpc.java`。

所以业务价值就一句话：**BMC 上各写各的进程，靠这一份定义保证"开机"在两个进程里是同一个枚举值、同一个属性名、同一个错误名。**

---

## 1. 真实场景一：面板上一个"开机"动作，是怎么被所有进程理解的

### 1.1 先把真实定义摆出来

`xyz.openbmc_project.State.Host` 这个接口的完整定义在 `yaml/xyz/openbmc_project/State/Host.interface.yaml`。它的开头和属性段长这样：

```yaml
description: >
    Implement to provide host state management          # :1-2

properties:                                              # :4
    - name: RequestedHostTransition                      # :5
      type: enum[self.Transition]                        # :6
      default: "Off"                                     # :7
      description: >                                     # :8-10
          The desired host transition.  This will be preserved across AC power
          cycles of the BMC.
      errors:                                            # :11
          - xyz.openbmc_project.State.Host.Error.BMCNotReady       # :12
          - xyz.openbmc_project.Common.Error.NotAllowed            # :13
          - xyz.openbmc_project.Common.Error.Unavailable           # :14

    - name: AllowedHostTransitions                       # :16
      type: set[enum[self.Transition]]                   # :17
      flags:                                             # :18
          - const                                        # :19

    - name: CurrentHostState                             # :26
      type: enum[self.HostState]                         # :27
      ...                                                # :28-32

    - name: RestartCause                                 # :34
      type: enum[self.RestartCause]                      # :35
      default: "Unknown"                                 # :36
```

枚举值集中列在同一个文件的 `enumerations:` 段（`yaml/xyz/openbmc_project/State/Host.interface.yaml:40`），"开机"要用的 `On` 在这里：

```yaml
- name: Transition                                       # :41
  values:
      - name: "Off"                                      # :45
      - name: "On"                                       # :48-50
      - name: "Reboot"                                   # :51
      - name: "GracefulWarmReboot"                       # :55
      - name: "ForceWarmReboot"                          # :60
                            —— yaml/xyz/openbmc_project/State/Host.interface.yaml:41-63
```

对象放在哪个路径，也写在同一个文件末尾：

```yaml
paths:                                                   # :138
    - namespace: /xyz/openbmc_project/state              # :139
      segments:
          - name: Host                                   # :141
            value: host                                  # :142
                            —— yaml/xyz/openbmc_project/State/Host.interface.yaml:138-142
```

### 1.2 完整链路（每一跳都带出处）

现在把"用户点开机"到"BMC 真的动起来"之间的链路画出来，每一步都指回上面这些定义：

```text
[用户/Redfish 请求：开机]
        │
        ▼
调用方要操作对象 /xyz/openbmc_project/state/host0
        │  路径从哪来？
        └──> paths.namespace + segments
             yaml/xyz/openbmc_project/State/Host.interface.yaml:138-142
             （namespace=/xyz/openbmc_project/state, segment Host→host）
        │
        ▼
写属性 RequestedHostTransition，值必须是枚举 Transition 里的一个
        │  属性名/类型从哪来？
        └──> properties[0].name/type/default
             yaml/xyz/openbmc_project/State/Host.interface.yaml:5-7
        │  "On" 这个合法取值从哪来？
        └──> enumerations[Transition].values[name=On]
             yaml/xyz/openbmc_project/State/Host.interface.yaml:48-50
        │
        ▼
实现方（提供 host 状态的对象）收到写入，按枚举做状态机
        │  它凭什么知道有这些状态？
        └──> 同一个文件里的 HostState 枚举
             yaml/xyz/openbmc_project/State/Host.interface.yaml:65-99
        │  当前状态从哪读？
        └──> CurrentHostState 属性（只读含义见 description）
             yaml/xyz/openbmc_project/State/Host.interface.yaml:26-32
        │
        ▼
如果 BMC 还没 Ready，拒绝这次写入，报 BMCNotReady
        │  错误名从哪来？
        └──> RequestedHostTransition.errors[0]
             yaml/xyz/openbmc_project/State/Host.interface.yaml:11-12
        │  这个错误本身的定义在哪？
        └──> yaml/xyz/openbmc_project/State/Host.errors.yaml:20
        │
        ▼
调用方和实现方两边都靠同一份生成头文件对齐类型
        │  头文件谁生成？
        └──> gen/xyz/openbmc_project/State/Host/meson.build:5-44
             输出 error.hpp / common.hpp / server.hpp / client.hpp
             （同文件 :11-19）
```

这条链路里有一个特别值得 Java 程序员注意的设计：**枚举值在线上传输的不是整数，而是名字字符串。** 这点在 `requirements.md` 里写明了：

```text
The sdbusplus implementation has built-in support for enumerations, which flow
across the dbus as uniquely encoded string values but has support in the
bindings for automatically converting to a C++ enum type.
                                        —— requirements.md:26-30
```

也就是说，D-Bus 线上跑的是 `"On"` 这个字符串，但生成的绑定会自动把它转成 C++ `enum class`。

> **Java 类比**：这就等于你在 protobuf 里写 `enum Transition { Off=0; On=1; }`，JSON 序列化出来是 `"On"`，Java 侧拿到的是类型安全的 `Transition.ON`。`enum[self.Transition]` 里的 `self.` 意思就是"在本接口的 `enumerations:` 段里找"，相当于 Java 里引用同一个类里定义的嵌套 enum。

### 1.3 这里要点出的概念（现在才点）

链路看完了，再回头给这堆东西起名字，就好懂了：

| 代码里出现的写法 | 出处 | 它是什么 |
| --- | --- | --- |
| `properties:` | `yaml/xyz/openbmc_project/State/Host.interface.yaml:4` | 一组可读/可写的属性，相当于 Java bean 的 getter/setter |
| `enumerations:` | `yaml/xyz/openbmc_project/State/Host.interface.yaml:40` | 枚举定义，相当于 Java `enum` |
| `flags: [const]` | `yaml/xyz/openbmc_project/State/Host.interface.yaml:18-19` | 常量属性，实现方不能随便改；`AllowedHostTransitions` 用它 |
| `flags: [readonly]` | `yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:56-57` | 只读属性，`UpdatedTime` 用它 |
| `errors:` | `yaml/xyz/openbmc_project/State/Host.interface.yaml:11-14` | 声明这个属性可能抛哪些错误 |
| `paths:` | `yaml/xyz/openbmc_project/State/Host.interface.yaml:138-142` | 告诉实现方对象应该挂到哪个 D-Bus 路径下 |

注意 `flags` 的两种值在真实代码里都出现了：`const` 在 `yaml/xyz/openbmc_project/State/Host.interface.yaml:19`，`readonly` 在 `yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:57`。术语照代码写，不要自己翻译成"不可变标志位"之类的新词。

#### 再看一眼 `HostState`：为什么它比"开机/关机"复杂

Scenario 里说"按枚举做状态机"，这个状态机到底有多少状态，还是得回到代码。把 `HostState` 完整摆出来：

```yaml
- name: HostState                                        # :65
  description: >
      The current state of the host firmware            # :66-67
  values:
      - name: "Off"                                      # :69
        description: Host firmware is not running        # :70-71
      - name: "TransitioningToOff"                       # :72
      - name: "Standby"                                  # :75
        description: >
            Host firmware is in a standby state. It is awaiting a Transition
            request to On to move to Running. This is an intermediate state
            that some implementations of this object may move to between Off
            and Running.                                 # :77-80
      - name: "Running"                                  # :81
      - name: "TransitioningToRunning"                   # :84
      - name: "Quiesced"                                 # :87
        description: >
            The host firmware is quiesced. The host firmware is enabled but
            either unresponsive or only processing a restricted set of
            commands. ...
      - name: "DiagnosticMode"                           # :94
        description: >
            The host firmware is capturing debug information. Powering off your
            system while the host is in this state will prevent the debug
            data from being properly collected. ...
                            —— yaml/xyz/openbmc_project/State/Host.interface.yaml:65-99
```

这里正好能说明"字典"的价值：`Quiesced`（`yaml/xyz/openbmc_project/State/Host.interface.yaml:87`）和 `DiagnosticMode`（`yaml/xyz/openbmc_project/State/Host.interface.yaml:94`）不是随便加的装饰，而是**对实现方的硬约束**——比如 `DiagnosticMode` 的描述明说此时断电会丢调试数据（`yaml/xyz/openbmc_project/State/Host.interface.yaml:96-99`）。调用方读 `CurrentHostState`（`yaml/xyz/openbmc_project/State/Host.interface.yaml:26-32`）时看到的是同一个枚举里的值，不需要猜"这个状态是不是还能开机"。

同样，`RequestedHostTransition` 的描述里特意强调这个值"会跨 BMC 交流电源周期保留"（`yaml/xyz/openbmc_project/State/Host.interface.yaml:8-10`）。这类语义只写在 YAML 的 `description` 里，生成不出代码，但它是业务规则的一部分——看代码之前先看这里，能省掉很多真机试错。

> **Java 类比**：把这些 `description` 想成接口方法上的 Javadoc，或者 `@apiNote`。编译器不检查它，但它是实现契约的一部分。openUBMC 里"规范文档"承担的角色，在这里就是这些 `description` 字段加上 `requirements.md`。

### 1.4 为什么"同一份定义"是刚需

假设没有这本字典：提供 host 状态的进程把属性叫 `RequestedTransition`，调用方写成 `RequestedHostTransition`，两边都能编译通过，但运行时永远对不上——而且这种错在 C/C++ 里往往要到真机联调才暴露。

有了这份 YAML 之后，两边都从它生成代码：

- 服务端拿 `server.hpp` / `aserver.hpp`（`gen/xyz/openbmc_project/State/Host/meson.build:15-17`）去实现；
- 客户端拿 `client.hpp`（`gen/xyz/openbmc_project/State/Host/meson.build:18`）去调用；
- 连错误码都有专门的 `error.hpp` / `error.cpp`（`gen/xyz/openbmc_project/State/Host/meson.build:12-13`）。

> **Java 类比**：这跟"前后端共用同一份 `.proto` 文件生成 stub"是一个道理。你不会在 Java 里手写两遍 DTO，OpenBMC 也不该在两个进程里手写两遍属性名。

---

## 2. 真实场景二：一条硬件告警，从产生到被看见

### 2.1 先把真实定义摆出来

第二个场景看日志条目 `xyz.openbmc_project.Logging.Entry`。它描述的是 BMC 里"一条错误日志"该长什么样：

```yaml
description: >
    Implement to provide event/error entry attributes. This interface should be
    instantiated for the phosphor::logging namespace. This interface is a
    collection of objects, therefore it is required to implement
    org.freedesktop.DBus.ObjectManager on the logging namespace root.    # :1-5
                            —— yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:1-7
```

这一段直接规定了两件事：这个接口是"一条条对象"的集合；根对象必须实现 `org.freedesktop.DBus.ObjectManager`（`yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:5`）。后者在本仓库里也有真实定义，见 `yaml/org/freedesktop/DBus/ObjectManager.interface.yaml:11-12`（`GetManagedObjects` 方法）。

一条日志带哪些字段，看属性段：

```yaml
properties:                                              # :8
    - name: Id              type: uint32                 # :9-10
    - name: Timestamp       type: uint64                 # :13-14
    - name: Severity        type: enum[self.Level]       # :17-18
    - name: Message         type: string                 # :21-22
    - name: EventId         type: string                 # :25-26
    - name: AdditionalData  type: dict[string, string]   # :33-34
    - name: Resolution      type: string                 # :37-38
    - name: Resolved        type: boolean                # :41-42
      errors:                                            # :48
          - xyz.openbmc_project.Common.Error.Unavailable # :49
    - name: ServiceProviderNotify                        # :50
      type: enum[self.Notify]                            # :51
      default: NotSupported                              # :52
    - name: UpdateTimestamp type: uint64                 # :59-60
                            —— yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:8-66
```

两个枚举也在这个文件里：

```yaml
enumerations:                                            # :68
    - name: Level                                        # :69
      values:                                            # :72
          - name: Emergency                              # :73
          - name: Alert                                  # :76
          - name: Critical                               # :79
          - name: Error                                  # :82
          - name: Warning                                # :85
          - name: Notice                                 # :88
          - name: Informational                          # :91
          - name: Debug                                  # :94
    - name: Notify                                       # :98
      values:
          - name: NotSupported                           # :102
          - name: Notify                                 # :105
          - name: Inhibit                                # :108
                            —— yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:68-110
```

方法段只有一个，但很有代表性：

```yaml
methods:                                                 # :112
    - name: GetEntry                                     # :113
      returns:                                           # :117
          - name: fd                                     # :118
            type: unixfd                                 # :119
                            —— yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:112-121
```

### 2.2 完整链路（每一跳都带出处）

```text
[某个服务检测到硬件故障，调日志 API 写一条 error]
        │
        ▼
日志服务创建一条 Entry 对象，比如 /xyz/openbmc_project/logging/entry/1
        │  路径从哪来？
        └──> paths: /xyz/openbmc_project/logging + Entry→entry
             yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:126-130
        │  谁来枚举这些对象？
        └──> description 要求根对象实现 ObjectManager
             yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:3-5
             GetManagedObjects 定义在
             yaml/org/freedesktop/DBus/ObjectManager.interface.yaml:11-12
        │
        ▼
这条 Entry 被填上字段：Id / Timestamp / Severity / Message / AdditionalData ...
        │  字段类型从哪来？
        └──> yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:9-66
        │  Severity 的取值从哪来？
        └──> Level 枚举 yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:69-96
        │  AdditionalData 为什么是字典？
        └──> dict[string, string] yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:33-34
        │       （description :35-36 说它是 {VAR,val} 形式的元数据）
        │
        ▼
调用方要看原始二进制日志内容 → 调 GetEntry 方法
        │  方法签名从哪来？
        └──> methods[GetEntry] returns fd: unixfd
             yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:112-121
        │
        ▼
这个接口默认由谁提供？
        └──> service_names: default: xyz.openbmc_project.Logging
             yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:123-124
        │
        ▼
"错误字典"本身还要分三类文件写：
        ├──> *.errors.yaml  ：简单错误名+说明
        │    例 yaml/xyz/openbmc_project/State/Host.errors.yaml:20 (BMCNotReady)
        ├──> *.events.yaml  ：带 severity / metadata / 多语言 message / errno
        │    例 yaml/xyz/openbmc_project/State/Power.events.yaml:3-18 (PowerRailFault)，version 在 :1
        └──> *.metadata.yaml：错误之间的继承关系
             例 yaml/xyz/openbmc_project/State/Host.metadata.yaml:1-7
        │
        ▼
events 文件会被生成成 Redfish Registry JSON
        └──> inst_registry_dir = datadir/redfish-registry/项目名
             gen/meson.build:17
             --command registry 生成 *.json，见
             gen/xyz/openbmc_project/State/meson.build:56-65
```

> **Java 类比**：`dict[string, string]` 就是 `Map<String,String>`；`unixfd` 可以想成 Java 里的方法返回 `FileChannel` 句柄——传的是"能读的文件描述符"，不是把文件内容整个序列化出来。`events.yaml` 里 `{PowerRail}` 这种占位符（`yaml/xyz/openbmc_project/State/Power.events.yaml` 中 `message:` 一行）就像 `MessageFormat.format("... {0}", rail)`。

### 2.3 这里要点出的概念（现在才点）

- **`methods`**：`yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:112-121` 的 `GetEntry` 是真正的方法定义，带 `returns`。
- **`returns` + `type: unixfd`**：`yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:117-119`。这是 D-Bus 特有的类型，Java 里没有直接对应物，按"文件句柄"理解最省事。
- **`service_names`**：`yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:123-124`。注意不是所有接口都有，它是给实现方的一个"默认服务名"提示。
- **错误分三层写**：`errors.yaml`（名字 + 说明）、`events.yaml`（结构化事件 + Redfish 用消息）、`metadata.yaml`（继承关系）。这三类的真实文件分别见 `yaml/xyz/openbmc_project/State/Host.errors.yaml:20`、`yaml/xyz/openbmc_project/State/Power.events.yaml:3-18`、`yaml/xyz/openbmc_project/State/Host.metadata.yaml:1-7`。

#### 三类错误文件到底差在哪，看真代码

`errors.yaml` 最简单，就是名字加一行说明：

```yaml
# xyz.openbmc_project.State.Host.Error.BMCNotReady        # :19
- name: BMCNotReady                                        # :20
  description: The BMC is not in a Ready state, no host operations allowed  # :21
                            —— yaml/xyz/openbmc_project/State/Host.errors.yaml:19-21
```

`events.yaml` 就结构化得多，`yaml/xyz/openbmc_project/State/Power.events.yaml` 开头带着 `version: 1.0.1`（`:1`），然后一个错误长这样：

```yaml
errors:                                                    # :3
    - name: PowerRailFault                                 # :4
      severity: error                                      # :5
      metadata:                                            # :6
          - name: PowerRail                                # :7
            type: object_path                              # :8
            primary: true                                  # :9
            description: The name or identifier of the power rail.  # :10
          - name: FailureData                              # :11
            type: string                                   # :12
      en:                                                  # :14
          description: A fault has been detected on a power rail.  # :15
          message: A fault has been detected on power rail '{PowerRail}'.  # :16
      errno: EIO                                           # :17
                            —— yaml/xyz/openbmc_project/State/Power.events.yaml:3-17
```

对照一下就很清楚：

| 文件类型 | 关键字段 | 给谁用 |
| --- | --- | --- |
| `*.errors.yaml` | `name` / `description` | 生成 D-Bus 错误类型，接口里的 `errors:` 引用它（如 `yaml/xyz/openbmc_project/State/Host.interface.yaml:12` → `yaml/xyz/openbmc_project/State/Host.errors.yaml:20`） |
| `*.events.yaml` | `severity` / `metadata` / `en.message` / `errno` / `version` | 生成事件日志条目和 Redfish Registry JSON（`gen/meson.build:17`、`gen/xyz/openbmc_project/State/meson.build:56-65`） |
| `*.metadata.yaml` | `name` / `inherits` | 声明错误之间的继承，如 `SoftOffTimeout` 继承 `xyz.openbmc_project.Common.Timeout`（`yaml/xyz/openbmc_project/State/Host.metadata.yaml:1-3`） |

`metadata` 里的 `primary: true`（`yaml/xyz/openbmc_project/State/Power.events.yaml:9`）表示哪个字段是事件的主对象；`{PowerRail}` 占位符（`yaml/xyz/openbmc_project/State/Power.events.yaml:16`）会在运行时被替换成实际值。这些字段都精确定义了"一条告警长什么样"，比一句自由文本 `description` 有用得多。

关于 Redfish：本仓库**没有**一个叫 `redfish-mapping.yaml` 的文件。它和 Redfish 的真实连接点是 `events.yaml` → `--command registry` → JSON → 安装到 `datadir/redfish-registry/` 这条构建链（`gen/meson.build:17`、`gen/xyz/openbmc_project/State/meson.build:56-65`）。D-Bus 属性到 Redfish 属性的一一映射不在这里维护，那属于消费方（如 bmcweb）的事。

---

## 3. 收尾：它和 sdbusplus / bmcweb / state-manager 的分工

把边界划清楚，才不会学混：

| 角色 | 干什么 | 本仓库里的锚点 |
| --- | --- | --- |
| **phosphor-dbus-interfaces（本仓库）** | 只写定义：属性/方法/信号/枚举/关联/路径；产出头文件 + 一个库 | `README.md:3-4`；`meson.build:77-85` 产出库；`gen/xyz/openbmc_project/State/Host/meson.build:11-19` 产出头文件 |
| **sdbusplus** | 提供 `sdbus++` 生成工具和 C++ 运行时绑定库 | `meson.build:15-17` 声明依赖与两个程序；`subprojects/sdbusplus.wrap:2-6` 说明从哪拉工具 |
| **服务实现方（如 state-manager / phosphor-logging）** | 按定义实现真正的功能，占住 service name、`implements` 服务端骨架 | 接口描述写"Implement to provide host state management"：`yaml/xyz/openbmc_project/State/Host.interface.yaml:1-2`；服务端骨架 `server.hpp`/`aserver.hpp` 在 `gen/xyz/openbmc_project/State/Host/meson.build:15-17`；默认日志服务名写在 `yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:123-124` |
| **调用方（如 bmcweb）** | 用生成的客户端绑定去读写属性、调方法；把 D-Bus 数据翻译成 Redfish/HTTP | 客户端绑定 `client.hpp` 在 `gen/xyz/openbmc_project/State/Host/meson.build:18`；Redfish Registry 来源在 `gen/meson.build:17` |

一句话记法：

```text
本仓库定规矩  →  sdbusplus 把规矩变成 C++ 类型  →  实现方按规矩干活  →  调用方按规矩来取
```

哪些目录会被真正生成，由 meson 选项决定：默认只开 `xyz/openbmc_project` 和 `org/freedesktop`（`README.md:21-22`），对应 `meson.build:45-53` 的 `data_org_freedesktop` 与 `data_xyz_openbmc_project`；`com/ibm`、`org/open_power` 要显式加 `-D`（`README.md:24-25`、`meson.build:33-34`、`meson.build:48-49`）。

下一篇《DBus接口机制讲解》会顺着这两个场景，把一份 YAML 具体怎么变成 `server.hpp` / `client.hpp`、信号和关联怎么生成、meson 怎么串起来，逐层拆开。
