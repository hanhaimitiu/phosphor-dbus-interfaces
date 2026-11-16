# DBus接口机制讲解：一份 YAML 是怎么变成 C++ 类的

> **写给谁**：有 openUBMC 经历、7 年没碰 C++ 的 Java 程序员，正在系统学 OpenBMC。
> **业务背景看哪篇**：先读同目录《DBus接口业务详解》，那里用"开机"和"记日志"两个真实场景讲清了这个仓库为什么存在。
> **本文回答什么问题**：`Sensor.Value.interface.yaml` 里几行 `properties:`，编译后到底变成了哪些 `.hpp` / `.cpp`？信号、方法、关联又是怎么进去的？meson 在其中穿了什么线？
> **讲法**：先拆一个完整例子，再回头讲目录、语法、生成器、各类成员和演进。所有 C++ 概念都用 Java 的注解处理器 / protobuf 生成代码类比。
> **路径说明**：下文 `path:行号` 均相对本仓库根目录 `phosphor-dbus-interfaces/`。

---

## 0. 先讲一个真实例子：`Sensor.Value` 从 YAML 到 5 个头文件

### 0.1 输入：真实的 YAML 片段

`yaml/xyz/openbmc_project/Sensor/Value.interface.yaml` 是一个只有 `properties`、没有 `methods` 和 `signals` 的"纯属性型"接口，最适合看生成机制。把它的属性段原样摆出来：

```yaml
properties:                                              # :29
    - name: Value                                        # :30
      type: double                                       # :31
      description: >
          The sensor reading.                            # :32-33

    - name: MaxValue                                     # :34
      type: double                                       # :35
      default: infinity                                  # :36

    - name: MinValue                                     # :39
      type: double                                       # :40
      default: -infinity                                 # :41

    - name: UpdatedTime                                  # :44
      type: uint64                                       # :45
      default: 0                                         # :46
      flags:                                             # :56
          - readonly                                     # :57

    - name: Unit                                         # :58
      type: enum[self.Unit]                              # :59
                            —— yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:29-59
```

`Unit` 的取值在同文件的枚举段：

```yaml
enumerations:                                            # :78
    - name: Unit                                         # :79
      values:
          - name: Amperes                                # :83
          - name: CFM                                    # :86
          - name: DegreesC                               # :89
          - name: Hertz                                  # :92
          - name: Joules                                 # :95
          - name: LPM                                    # :98
          - name: AmpereHours                            # :101
          - name: Meters                                 # :104
          - name: Percent                                # :107
          - name: PercentRH                              # :110
          - name: Pascals                                # :113
          - name: Radians                                # :116
          - name: RPMS                                   # :119
          - name: Volts                                  # :122
          - name: Watts                                  # :125
                            —— yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:78-128
```

还有路径段，告诉实现方"对象要挂到哪里"：

```yaml
paths:                                                   # :145
    - namespace: /xyz/openbmc_project/sensors            # :146
      segments:
          - name: Temperature                            # :172
            value: temperature                           # :173
          - name: Voltage                                # :181
            value: voltage                               # :182
                            —— yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:145-182
```

### 0.2 中间：meson 怎么描述这次生成

生成动作不在本仓库的 Python 脚本里，而在一个 meson `custom_target` 中。文件是 `gen/xyz/openbmc_project/Sensor/Value/meson.build`（注意第一行的 `# Generated file; do not modify.`，`:1`）：

```meson
sdbusplus_current_path = 'xyz/openbmc_project/Sensor/Value'   # :3

generated_sources += custom_target(                          # :5
    'xyz/openbmc_project/Sensor/Value__cpp'.underscorify(),  # :6
    input: [                                                  # :7
        '../../../../../yaml/xyz/openbmc_project/Sensor/Value.interface.yaml',  # :8
    ],
    output: [                                                 # :10
        'common.hpp',                                         # :11
        'server.hpp',                                         # :12
        'server.cpp',                                         # :13
        'aserver.hpp',                                        # :14
        'client.hpp',                                         # :15
    ],
    depend_files: sdbusplusplus_depfiles,                    # :17
    command: [                                                # :18
        sdbuspp_gen_meson_prog,                               # :19
        '--command', 'cpp',                                   # :20-21
        '--output', meson.current_build_dir(),                # :22-23
        '--tool', sdbusplusplus_prog,                         # :24-25
        '--directory', meson.current_source_dir() / '../../../../../yaml',  # :26-27
        'xyz/openbmc_project/Sensor/Value',                   # :28
    ],
    install: should_generate_cpp,                            # :30
    install_dir: [                                           # :31
        get_option('includedir') / sdbusplus_current_path,    # :32
        get_option('includedir') / sdbusplus_current_path,    # :33
        false,                                               # :34  server.cpp 不安装
        get_option('includedir') / sdbusplus_current_path,    # :35
        get_option('includedir') / sdbusplus_current_path,    # :36
    ],
    build_by_default: should_generate_cpp,                   # :38
)
                            —— gen/xyz/openbmc_project/Sensor/Value/meson.build:1-39
```

### 0.3 输出：5 个文件各自干什么

从 `output:` 列表（`gen/xyz/openbmc_project/Sensor/Value/meson.build:10-16`）可以看出一次生成会吐出 5 个文件。按 Java 经验对应过去：

| 生成文件 | 出处 | 按 Java 经验理解 |
| --- | --- | --- |
| `common.hpp` | `gen/xyz/openbmc_project/Sensor/Value/meson.build:11` | 公共类型定义：`Unit` 枚举、属性类型别名。相当于 protobuf 生成的 `XxxOuterClass`，所有类型都在这儿 |
| `server.hpp` | `:12` | 服务端要继承的接口基类，声明每个属性的 getter/setter。相当于你 `implements` 的 Java interface |
| `server.cpp` | `:13` | 把 C++ 调用转成 D-Bus 消息的代码。相当于 gRPC 生成代码里的服务端 stub（只在编译期用，`install_dir` 是 `false`，`:34`） |
| `aserver.hpp` | `:14` | "异步"版本的服务端基类。Java 里没有直接对应物，可理解为带 `CompletableFuture` 风格的另一套接口 |
| `client.hpp` | `:15` | 客户端拿着调属性的代理类。相当于 protobuf/gRPC 生成的 blocking stub |

> **Java 类比（重点）**：整件事就是**注解处理器 / protobuf 代码生成**。
> 你写 `@Interface("xyz.openbmc_project.Sensor.Value")` 式的元数据（这里是 YAML），注解处理器在编译期扫描它（这里是 `sdbus++ --command cpp`，`gen/xyz/openbmc_project/Sensor/Value/meson.build:20-28`），生成 `XxxGrpc.java`（这里是 `server.hpp` + `client.hpp`）。区别只是：Java 的处理器是 `.java` 实现的，这里的生成器是独立的 C++ 工具 `sdbus++`，由 `meson.build:16` 用 `find_program('sdbus++', native: true)` 找到。

### 0.4 一个有信息量的差异：为什么 State.Host 多两个文件

对比 `gen/xyz/openbmc_project/State/Host/meson.build:5-44`，它的 `output:` 是 7 个，多了 `error.cpp` 和 `error.hpp`（`:12-13`）。原因就在 `input:`：

```meson
input: [
    '../../../../../yaml/xyz/openbmc_project/State/Host.errors.yaml',   # :8
    '../../../../../yaml/xyz/openbmc_project/State/Host.interface.yaml', # :9
],
                            —— gen/xyz/openbmc_project/State/Host/meson.build:7-10
```

而 `Sensor.Value` 的 `input:` 只有 `Value.interface.yaml` 一个文件（`gen/xyz/openbmc_project/Sensor/Value/meson.build:7-9`）。**结论：只要接口目录旁边有同名 `.errors.yaml`，生成物就多出 `error.hpp` / `error.cpp`。** 这是一条能直接拿去验证的规则，不用记。

---

## 1. 先看目录与构建关系

### 1.1 `yaml/`：按反域名分层，和 Java 包如出一辙

顶层只有三个目录：`com/`、`org/`、`xyz/`。默认只有 `xyz/openbmc_project` 和 `org/freedesktop` 会参与构建（`README.md:21-22`），对应 `meson.build:45-53`：

```meson
if get_option('data_org_freedesktop')
    yaml_selected_subdirs += 'org/freedesktop'      # meson.build:45-47
endif
if get_option('data_xyz_openbmc_project')
    yaml_selected_subdirs += 'xyz/openbmc_project'  # meson.build:51-53
endif
```

`xyz/openbmc_project/` 下面继续按功能分：`State/`、`Sensor/`、`Logging/`、`Inventory/`、`Chassis/`、`Software/`、`Network/`……（均为 `yaml/xyz/openbmc_project/` 下的真实目录名）。

> **Java 类比**：`xyz.openbmc_project.State.Host` ↔ 包 `xyz.openbmc_project.state` 下的 `Host` 类型。反域名写法的目的完全一样：避免不同厂商撞名。`com/ibm`、`com/intel`、`com/qualcomm`、`org/open_power` 这些目录就是"厂商自己的包"，要用 `-Ddata_*` 打开（`meson.build:27-49`、`README.md:24-27`）。

### 1.2 一个接口目录里能出现 4 种文件后缀

全仓库统计：`*.interface.yaml` 347 个、`*.errors.yaml` 39 个、`*.events.yaml` 24 个、`*.metadata.yaml` 27 个。以 `State/Host` 为例，它有三个文件：

```text
yaml/xyz/openbmc_project/State/Host.interface.yaml    ← 接口本体
yaml/xyz/openbmc_project/State/Host.errors.yaml       ← 错误名清单
yaml/xyz/openbmc_project/State/Host.metadata.yaml     ← 错误继承关系
```

`yaml/xyz/openbmc_project/State/Host.interface.yaml:11-14` 的 `errors:` 引用 `BMCNotReady`，名字本身定义在 `yaml/xyz/openbmc_project/State/Host.errors.yaml:20`；`yaml/xyz/openbmc_project/State/Host.metadata.yaml:1-3` 又说 `SoftOffTimeout` 继承 `xyz.openbmc_project.Common.Timeout`。**同一个 basename + 不同后缀 = 同一组相关定义**，生成器靠 basename 把它们配对（从 `gen/xyz/openbmc_project/State/Host/meson.build:8-9` 两个 `input:` 用的是同一个 `Host` 前缀可以看出）。

### 1.3 `gen/`：不是脚本目录，是"生成的 meson 文件"目录

这里和很多人的直觉相反。`gen/README` 只有两行：

```text
This directory contains generated meson.build files from sdbus++-gen-meson.
Do not edit them by hand.
                                        —— gen/README:1-2
```

也就是说：`gen/` 下放的是**已经生成好的 meson 片段**，不是生成器源码。`gen/meson.build:1` 也标了 `# Generated file; do not modify.`

真正生成这些片段的命令在 `gen/regenerate-meson:22-27`：

```bash
$SDBUSPP_GEN_MESON \
    --command meson \
    --directory "$REPO_ROOT/yaml" \
    --output "$REPO_ROOT/gen"
                            —— gen/regenerate-meson:22-26
```

工具从哪来？`gen/regenerate-meson:6-20` 的查找顺序是：环境变量 `SDBUSPP_GEN_MESON` → `$PATH` → `subprojects/sdbusplus/tools/sdbus++-gen-meson`。而 subproject 的来源写在 `subprojects/sdbusplus.wrap:2-6`（`url = https://github.com/openbmc/sdbusplus.git`，`program_names = sdbus++, sdbus++-gen-meson`）。

> **注意**：本仓库里**没有** `sdbusplus_gen.py` / `yaml_gen.py` 这种脚本，也没有任何 `*.gen.hpp` 文件（全仓库 `find` 结果为 0）。生成逻辑住在 sdbusplus 仓库，本仓库只保存 YAML 和"生成的 meson 文件"。文档里不要写出不存在的本地路径。

### 1.4 顶层 `meson.build` 的职责表

| 行 | 做了什么 |
| --- | --- |
| `meson.build:1-13` | 声明工程用 C++23（`cpp_std=c++23`，`:7`），`werror=true`（`:9`） |
| `meson.build:15` | `dependency('sdbusplus')` 声明运行时依赖 |
| `meson.build:16-17` | 找到两个 native 程序：`sdbus++` 和 `sdbus++-gen-meson` |
| `meson.build:25-53` | 根据 `data_*` 选项拼出 `yaml_selected_subdirs` |
| `meson.build:56-63` | 把选中的 YAML 原样安装到 `datadir/phosphor-dbus-yaml` |
| `meson.build:65-67` | 三个开关：`should_generate_cpp` / `should_generate_markdown` / `should_generate_registry` |
| `meson.build:70` | `subdir('gen')`，把生成规则引进来 |
| `meson.build:74` | `subdir('registry')`，处理 registry 文件 |
| `meson.build:77-85` | 用 `generated_sources` 编出 `libphosphor_dbus` |
| `meson.build:87-96` | 生成 pkg-config，暴露 `yamldir` 和 `registry_dir` |
| `meson.build:98-107` | `declare_dependency`，把头文件和库一起交给下游 |

`gen/meson.build` 接手后做了三件关键事：

```meson
if sdbuspp_gen_meson_ver != 'sdbus++-gen-meson version 10'   # :8
    warning('Generated meson files from wrong version of sdbus++-gen-meson.')  # :9
endif
...
generated_sources = []      # :19
generated_markdown = []     # :20
generated_registry = []     # :21
foreach d : yaml_selected_subdirs
    subdir(d)               # :24  逐个进入 gen/<包路径>/meson.build
endforeach
...
if f.full_path().endswith('.hpp')   # :30
    generated_headers += f          # :31
endif
                            —— gen/meson.build:8-34
```

注意 `:8` 的版本校验：生成器版本必须匹配 `sdbus++-gen-meson version 10`，否则只是 warning。这是"生成物可能和工具不同步"的兜底提示。

---

## 2. YAML 语法：一个接口文件里能写哪些段

### 2.1 真实接口里出现过的段

把三篇代表接口里用过的顶层 key 列出来（都能在对应文件里逐行找到）：

| 顶层 key | 例子出处 | 作用 |
| --- | --- | --- |
| `description` | `yaml/xyz/openbmc_project/State/Host.interface.yaml:1-2` | 接口整体说明，会进生成的文档 |
| `properties` | `yaml/xyz/openbmc_project/State/Host.interface.yaml:4` | 属性列表 |
| `enumerations` | `yaml/xyz/openbmc_project/State/Host.interface.yaml:40` | 枚举定义 |
| `methods` | `yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:112` | 方法定义 |
| `signals` | `yaml/org/freedesktop/DBus/ObjectManager.interface.yaml:48` | 信号定义 |
| `associations` | `yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:129` | 对象间关联 |
| `service_names` | `yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:123` | 默认服务名提示 |
| `paths` | `yaml/xyz/openbmc_project/State/Host.interface.yaml:138` | 对象路径规划 |

### 2.2 类型系统：写在 `type:` 后面的那些方括号

这部分最容易劝退 Java 程序员，但对照着看其实很直白：

| YAML 写法 | 出现位置 | Java 里的对应 |
| --- | --- | --- |
| `double` / `uint32` / `uint64` / `boolean` / `string` | `yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:31`、`:45`；`yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:10`、`:42` | `double` / `int` / `long` / `boolean` / `String` |
| `enum[self.Unit]` | `yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:59` | 本接口里定义的嵌套 `enum Unit` |
| `set[enum[self.Transition]]` | `yaml/xyz/openbmc_project/State/Host.interface.yaml:17` | `Set<Transition>` |
| `dict[string, string]` | `yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:34` | `Map<String,String>` |
| `array[object_path]` | `yaml/xyz/openbmc_project/Association.interface.yaml:11` | `List<DBusObjectPath>` |
| `unixfd` | `yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:119` | 文件描述符，近似 `FileChannel` |
| `variant[...]` | `yaml/org/freedesktop/DBus/ObjectManager.interface.yaml:20` | 带类型标记的"任意类型"，近似 `Object` + 类型信息 |

`ObjectManager` 的 `GetManagedObjects` 返回类型是一个巨大的 `dict[...variant[...]]`（`yaml/org/freedesktop/DBus/ObjectManager.interface.yaml:19-46`）。这一段放到最后看就行——它是"把所有已知类型都列进 variant 联合类型"的写法，不是给人类读的。

### 2.3 写接口时的"规矩"也在这个仓库里

`requirements.md` 不是代码，但它是写 YAML 时的约束，而且直接解释了为什么某些类型长成现在这样：

```text
For countable entities always prefer `size` ... For non-countable values prefer
`uint64` or `int64`.                                    —— requirements.md:15-17
...
The sdbusplus implementation has built-in support for enumerations, which flow
across the dbus as uniquely encoded string values but has support in the
bindings for automatically converting to a C++ enum type.
                                        —— requirements.md:26-30
```

翻译成人话：不要为了省几个字节乱用 `uint8`（`requirements.md:11-13`）；能用枚举就别用裸字符串（`requirements.md:19-24` 说随意字符串只给人看、不能被代码解析）。这解释了为什么 `Sensor.Value` 的 `Unit` 是 `enum[self.Unit]`（`yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:59`）而不是 `string`。

---

## 3. 生成机制：从 YAML → `server.hpp` 的完整链路

### 3.1 五步串起来

```text
① 选目录
   meson.build:25-53 → yaml_selected_subdirs
        │
        ▼
② 找工具
   meson.build:16-17 → sdbus++ / sdbus++-gen-meson
   subprojects/sdbusplus.wrap:2-6 → 从 sdbusplus 仓库拉
        │
        ▼
③ 读"生成的 meson 文件"
   meson.build:70 → subdir('gen')
   gen/meson.build:23-25 → foreach subdir(d)
   gen/xyz/openbmc_project/Sensor/Value/meson.build:5-39 → custom_target
        │
        ▼
④ 跑生成器
   gen/xyz/openbmc_project/Sensor/Value/meson.build:20-28
   sdbus++ --command cpp --tool sdbus++ --directory <yaml 根> xyz/openbmc_project/Sensor/Value
        │
        ▼
⑤ 收集产物
   gen/meson.build:19-21 → generated_sources / markdown / registry
   gen/meson.build:27-34 → 从 *.hpp 收集 generated_headers
   meson.build:77-85 → 编进 libphosphor_dbus
   meson.build:98-107 → 作为 dependency 交给下游服务
```

### 3.2 三条并行的生成线

`custom_target` 的 `--command` 参数决定生成什么，同一个接口可能出现在三条线里：

| `--command` | 出处 | 产物 | 安装位置 |
| --- | --- | --- | --- |
| `cpp` | `gen/xyz/openbmc_project/State/Host/meson.build:24`、`gen/xyz/openbmc_project/Sensor/Value/meson.build:21` | `error.hpp` / `common.hpp` / `server.hpp` / `server.cpp` / `aserver.hpp` / `client.hpp`（`State/Host` 的 `:11-19`） | `includedir/<包路径>`，见 `:34-42` |
| `markdown` | `gen/xyz/openbmc_project/State/meson.build:40-41`、`:354-355` | `BMC.md` / `Host.md` 等（`:36`、`:350`） | `datadir/doc/<项目名>/<包路径>`，见 `gen/meson.build:16` |
| `registry` | `gen/xyz/openbmc_project/State/meson.build:64-65` | `BMC.json` 等 Redfish Registry | `datadir/redfish-registry/<项目名>/<包路径>`，见 `gen/meson.build:17` |

`markdown` 和 `registry` 的开关分别是 `should_generate_markdown` 和 `should_generate_registry`（`meson.build:66-67`）。`registry` 产物最后被 `registry/meson.build:1-6` 的 `cp -v --parents` 拷到单独目录，方便作为 subproject 时被依赖方使用。

### 3.3 "生成代码"为什么是可信的

因为生成器本身就是依赖的一部分：`meson.build:15` 声明 `dependency('sdbusplus')`，`meson.build:19-23` 在 sdbusplus 作为 subproject 时还会拿到 `sdbusplusplus_depfiles`，并把它作为所有 `custom_target` 的 `depend_files`（如 `gen/xyz/openbmc_project/Sensor/Value/meson.build:17`）。**生成器一变，所有生成目标都会重跑**——这正是注解处理器的语义。

---

## 4. property / signal / method 分别怎么落进去

### 4.1 property：最普通也最多

`yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:29-76` 有 5 个属性。除了 `type` 和 `description`，真正影响生成代码的是两个字段：

- `default:`：如 `MaxValue` 的 `infinity`（`:36`）、`MinValue` 的 `-infinity`（`:41`）、`UpdatedTime` 的 `0`（`:46`）。相当于 Java 字段的初始化值。
- `flags:`：`readonly`（`:57`）或 `const`（`yaml/xyz/openbmc_project/State/Host.interface.yaml:19`）。前者表示客户端不能写，后者表示连实现方都不该改。

还有 `errors:`，声明这个属性在被写时可能抛什么错，如 `Resolved` 属性抛 `xyz.openbmc_project.Common.Error.Unavailable`（`yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:48-49`）。

> **Java 类比**：一个 property ≈ 一对 getter/setter + 字段初始值 + `@Throws` 注解。`readonly` 就是只有 getter 没有 setter。

### 4.2 signal：看 ObjectManager 的两个信号

`yaml/org/freedesktop/DBus/ObjectManager.interface.yaml:48-95` 定义了两个信号：

```yaml
signals:                                                 # :48
    - name: InterfacesAdded                              # :49
      properties:                                        # :53
          - name: ObjectPath                             # :54
            type: object_path                            # :55
          - name: InterfacesAndProperties                # :58
    - name: InterfacesRemoved                            # :86
      properties:
          - name: ObjectPath                             # :90
            type: object_path                            # :91
          - name: Interfaces                             # :94
            type: array[string]                          # :95
                            —— yaml/org/freedesktop/DBus/ObjectManager.interface.yaml:48-95
```

注意一个术语细节：`signals:` 下面的参数列表在 YAML 里也叫 `properties:`（`:53`），但它的意思是"信号携带的数据"，不是对象的属性。代码里叫什么就写什么，别改口叫"参数列表"。

> **Java 类比**：signal ≈ 事件监听器。服务端 `emit InterfacesAdded(...)` 相当于 `listeners.forEach(l -> l.onEvent(event))`；客户端生成的头里提供注册/回调入口。`InterfacesAdded` 描述的是"对象上多了接口"（`:49`的描述行），和 Spring 里发布一个 `ApplicationEvent` 是同一回事。

### 4.3 method：看 GetEntry 和 GetManagedObjects

两个例子：

```yaml
# Logging.Entry
methods:                                                 # :112
    - name: GetEntry                                     # :113
      returns:
          - name: fd                                     # :118
            type: unixfd                                 # :119
                            —— yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:112-121

# ObjectManager
methods:                                                 # :11
    - name: GetManagedObjects                            # :12
      returns:
          - name: Objects                                # :18
            type: dict[object_path, dict[string, dict[string, variant[...]]]]
                            —— yaml/org/freedesktop/DBus/ObjectManager.interface.yaml:11-46
```

方法由 `name` + `returns`（+ 可选的入参）组成。`unixfd` 这种类型 Java 没有，按"返回文件句柄"理解即可。

> **Java 类比**：method ≈ interface 里的方法声明，`returns:` ≈ 返回类型。生成出来的 `client.hpp` 里会有一个同步调用方法，`server.hpp` 里有一个等着你 `override` 的虚方法——和 gRPC 生成的 `blockingStub` / `ImplBase` 一一对应。

---

## 5. association：D-Bus 对象之间怎么连

### 5.1 真实定义

`Sensor.Value` 末尾声明了两条关联：

```yaml
associations:                                            # :129
    - name: inventory                                    # :130
      reverse_name: sensors                              # :134
      required_endpoint_interfaces:                      # :135
          - xyz.openbmc_project.Inventory.Item           # :136
    - name: monitoring                                   # :137
      reverse_name: monitored_by                         # :141
      required_endpoint_interfaces:                      # :142
          - xyz.openbmc_project.Inventory.Item.Bmc       # :143
                            —— yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:129-143
```

而关联本身作为一种 D-Bus 接口，定义在 `yaml/xyz/openbmc_project/Association.interface.yaml`：

```yaml
description: >
    An object implementing Association describes a one way, one to many
    association where the parent object is associated to the objects in the
    Endpoints property, with the association type defined as the name of the
    object implementing Association.                     # :5-8
properties:                                              # :9
    - name: endpoints                                    # :10
      type: array[object_path]                           # :11
                            —— yaml/xyz/openbmc_project/Association.interface.yaml:1-13
```

翻译：`Association` 接口只有一个 `endpoints` 属性，类型是 `array[object_path]`，也就是"一串对象路径"。关系的类型由"实现这个关联接口的对象名"决定（`yaml/xyz/openbmc_project/Association.interface.yaml:6-8`）。

### 5.2 命名规矩写在 requirements.md

```text
Directed associations ... should be considered to have a "primary" and
"secondary" end ... a chassis might be "containing" (primary) any number of
other inventory objects which are "contained_by" (secondary) ...
                                        —— requirements.md:62-67
```

并且要求主端用现在分词 `-ing`、从端用过去分词 `-ed`（`requirements.md:90-94`）。回看 `Sensor.Value`：主端名 `inventory` 不符合 `-ing`，但 `monitoring` / `monitored_by` 是标准的一对（`yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:137-143`）——说明规矩也有历史遗留，按代码现状理解即可。

> **Java 类比**：关联 ≈ 对象上的 `List<其他对象引用>`，只不过存的是 D-Bus 路径字符串而不是内存引用。`required_endpoint_interfaces`（`yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:135-136`）相当于给这个引用加了个 `instanceof` 约束。`reverse_name` 就是双向关系里另一边的字段名，类似 JPA 的 `mappedBy`。

---

## 6. 红鱼映射（Redfish）：真实连接点在哪

先把结论摆前面：**这个仓库里没有 `redfish-mapping.yaml` 这类文件**。它和 Redfish 的连接是通过 `events.yaml` → Registry JSON 这条构建链实现的，不是通过属性映射表。

### 6.1 链路

```text
events.yaml（带 version / severity / en.message / errno）
   例 yaml/xyz/openbmc_project/State/Power.events.yaml:1-18
        │
        ▼
gen/<包路径>/meson.build 里的 --command registry
   例 gen/xyz/openbmc_project/State/meson.build:64-65
        │
        ▼
生成 *.json（如 BMC.json，gen/xyz/openbmc_project/State/meson.build:...）
        │
        ▼
安装到 datadir/redfish-registry/<项目名>/<包路径>
   gen/meson.build:17
        │
        ▼
registry/meson.build:1-6 用 cp --parents 拷到单独目录，方便 subproject 依赖
```

`yaml/xyz/openbmc_project/State/Power.events.yaml:16` 的 `message: A fault has been detected on power rail '{PowerRail}'.` 就是会进 Registry 的文本模板，`errno: EIO`（`:17`）是 errno 映射。`version: 1.0.1`（`:1`）是 Registry 自身的版本号。

### 6.2 谁来做"属性 → Redfish 字段"的映射

不在这个仓库。本仓库提供的只是被消费的素材：

- 生成的 `client.hpp`（`gen/xyz/openbmc_project/Sensor/Value/meson.build:15`）供调用方读写属性；
- 生成的 Registry JSON（`gen/meson.build:17`）供 Redfish 服务引用消息定义；
- YAML 本身被原样安装到 `datadir/phosphor-dbus-yaml`（`meson.build:56-63`），并在 pkg-config 里以 `yamldir` 暴露（`meson.build:93`、`meson.build:104`）。

调用方拿到 `yamldir` 就能知道去哪里找接口定义——这是"映射"能发生的物理前提，但映射逻辑本身由消费方维护。

---

## 7. 版本演进：从哪儿看出"接口会变"

### 7.1 三层版本

```text
工程版本：version: '1.0.0'                 meson.build:12
生成器版本：sdbus++-gen-meson version 10     gen/meson.build:8-11
事件版本：version: 1.0.1                    yaml/xyz/openbmc_project/State/Power.events.yaml:1
```

事件版本在不同文件里并不一致，这是真实存在的演进痕迹：

| 文件 | 版本 | 出处 |
| --- | --- | --- |
| `Common.events.yaml` | `1.0.0` | `yaml/xyz/openbmc_project/Common.events.yaml:1` |
| `State/BMC.events.yaml` | `1.2.0` | `yaml/xyz/openbmc_project/State/BMC.events.yaml:1` |
| `State/Fan.events.yaml` | `1.1.0` | `yaml/xyz/openbmc_project/State/Fan.events.yaml:1` |
| `Sensor/Threshold.events.yaml` | `1.2.0` | `yaml/xyz/openbmc_project/Sensor/Threshold.events.yaml:1` |
| `Software/Update.events.yaml` | `1.3.0` | `yaml/xyz/openbmc_project/Software/Update.events.yaml:1` |

也就是说，每个事件文件自己的 `version:` 是独立演进的，加一条错误、改一次消息模板都可以往上提。

### 7.2 兼容性靠什么保住

- **`default:` 给新属性兜底**：`UpdatedTime` 的 `default: 0`（`yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:46`），description 明说客户端可以把 `0` 读成"不支持"（`:52-53`）。老实现不写这个属性，新客户端也不会崩。
- **`flags: readonly` / `const` 锁定语义**：`yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:57`、`yaml/xyz/openbmc_project/State/Host.interface.yaml:19`。
- **枚举而不是整数**：`requirements.md:26-30` 说明枚举在线上是字符串，加一个新值不会让旧客户端把数字解释错。
- **生成器版本校验**：`gen/meson.build:8-14`，工具对不上会告警，防止"新 YAML + 老生成器"悄悄产出错代码。

> **Java 类比**：这整套就是"向后兼容的 DTO 设计"——新字段给默认值、不删老字段、枚举用名字不用序号，和 protobuf 的兼容原则一模一样。

---

## 8. 结尾：读一份陌生接口的最短路径

下次遇到一个没看过的 `*.interface.yaml`，按这个顺序读，基本不会迷路：

```text
① description                → 这个接口干嘛的（如 yaml/xyz/openbmc_project/State/Host.interface.yaml:1-2）
② properties + flags/default → 有哪些数据、能不能写（如 yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:29-76）
③ enumerations               → 枚举值有哪些（如 yaml/xyz/openbmc_project/State/Host.interface.yaml:40-136）
④ methods / signals          → 能调什么、会发什么（如 yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:112、ObjectManager:48）
⑤ associations               → 和谁有关联（如 yaml/xyz/openbmc_project/Sensor/Value.interface.yaml:129-143）
⑥ paths + service_names      → 对象在哪、默认谁提供（如 yaml/xyz/openbmc_project/Logging/Entry.interface.yaml:123-130）
⑦ 同 basename 的 errors/events/metadata → 错误怎么定义（如 yaml/xyz/openbmc_project/State/Host.errors.yaml:20）
⑧ gen/<同路径>/meson.build   → 实际生成哪些文件（如 gen/xyz/openbmc_project/Sensor/Value/meson.build:10-16）
```

一条底层心法：**这个仓库里所有东西都是"声明"**。`description` 是给人看的声明，`properties` / `methods` / `signals` 是给生成器看的声明，`paths` 是给实现方看的声明。真正跑起来的行为在别的仓库里，但它们的形状都从这里长出来——和 openUBMC 里"先有规范、后有实现"是同一种工作方式。
