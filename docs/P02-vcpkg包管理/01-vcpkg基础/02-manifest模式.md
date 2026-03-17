# 02-manifest 模式
> **所属模块：** P02-vcpkg包管理
> **前置知识：** [01-vcpkg基础概念.md](./01-vcpkg基础概念.md)
> **预计阅读时间：** 22 分钟
## 本节目标
读完本节后，你将能够：
1. 区分 Classic 模式与 Manifest 模式。
2. 编写包含核心字段的 `vcpkg.json`。
3. 理解 `vcpkg-configuration.json` 的关键配置。
4. 使用版本约束与 features 控制依赖。
5. 理解 configure 阶段自动安装机制。
6. 分析 KrKr2 的真实清单配置。

## 1. Manifest 模式 vs Classic 模式
vcpkg 有两种常见工作方式：Classic 与 Manifest。
二者都能安装包，但工程价值差异明显。

### 1.1 Classic 模式
Classic 模式依赖手工命令：
```bash
vcpkg install fmt
vcpkg install spdlog
```
优点是上手快、临时验证方便。
缺点是依赖信息容易散落在命令历史中。
多人协作时，版本漂移问题很常见。

### 1.2 Manifest 模式
Manifest 模式通过 `vcpkg.json` 声明依赖。
依赖配置成为项目仓库的一部分。
构建时读取清单并自动解析依赖。
这让团队协作与 CI 复现更可靠。

### 1.3 详细对比
| 维度 | Classic | Manifest |
|---|---|---|
| 依赖声明位置 | 命令历史 | `vcpkg.json` |
| 团队可见性 | 低 | 高 |
| 版本可追踪性 | 弱 | 强 |
| 可重复构建 | 弱 | 强 |
| CI 集成成本 | 较高 | 较低 |
| 项目隔离性 | 弱 | 强 |

结论：长期项目应优先 Manifest。

## 2. `vcpkg.json` 完整字段说明
`vcpkg.json` 是 Manifest 的核心。
下面覆盖最关键字段。

### 2.1 最小模板
```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg-tool/main/docs/vcpkg.schema.json",
  "name": "demo-project",
  "version-string": "0.1.0",
  "dependencies": [
    "fmt"
  ]
}
```

### 2.2 `name`
项目名称字段。
建议使用小写和连字符，便于统一。

### 2.3 `version` 与 `version-string`
- `version`：语义化版本。
- `version-string`：字符串更灵活。
KrKr2 当前使用 `version-string`。

### 2.4 `dependencies`
支持字符串与对象两种写法。

字符串写法：
```json
{
  "dependencies": [
    "fmt",
    "spdlog"
  ]
}
```

对象写法：
```json
{
  "dependencies": [
    {
      "name": "catch2",
      "default-features": false,
      "platform": "!android & !ios"
    }
  ]
}
```

### 2.5 `features`
项目级功能开关，可绑定额外依赖。

```json
{
  "name": "demo-project",
  "version-string": "0.1.0",
  "features": {
    "with-tests": {
      "description": "启用测试依赖",
      "dependencies": [
        "catch2"
      ]
    }
  }
}
```

### 2.6 `overrides`
用于强制固定某些依赖版本。

```json
{
  "dependencies": [
    "opencv4"
  ],
  "overrides": [
    {
      "name": "opencv4",
      "version": "4.7.0#6"
    }
  ]
}
```

### 2.7 `builtin-baseline`
用于锁定默认仓库版本基线。

```json
{
  "name": "demo-project",
  "version-string": "0.1.0",
  "builtin-baseline": "b1e15efef6758eaa0beb0a8732cfa66f6a68a81d",
  "dependencies": [
    "fmt"
  ]
}
```

## 3. `vcpkg-configuration.json` 详解
该文件控制依赖来源与注册表行为。

KrKr2 当前配置：
```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg-tool/main/docs/vcpkg-configuration.schema.json",
  "default-registry": {
    "kind": "builtin",
    "baseline": "b1e15efef6758eaa0beb0a8732cfa66f6a68a81d"
  },
  "overlay-ports": [
    "./vcpkg/ports"
  ],
  "overlay-triplets": [
    "./vcpkg/triplets"
  ]
}
```

### 3.1 `default-registry`
定义默认包来源与 baseline。

### 3.2 `registries`
用于追加私有或额外仓库。
常见写法是声明 `kind: git`，并通过 `packages` 指定由该私有仓库负责解析的包名集合。

### 3.3 `overlay-ports`
用于覆盖官方 ports 或追加本地 ports。
KrKr2 使用它维护项目特定移植规则。

## 4. 版本约束语法
Manifest 依赖对象支持版本范围控制。

| 语法 | 含义 | 示例 |
|---|---|---|
| `version>=` | 最低版本 | `"version>=": "10.0.0"` |
| `version<` | 严格上界 | `"version<": "2.0.0"` |
| `version=` | 精确版本 | `"version=": "1.2.3"` |

示例：
```json
{
  "dependencies": [
    {
      "name": "fmt",
      "version>=": "10.1.0"
    },
    {
      "name": "spdlog",
      "version<": "2.0.0"
    }
  ]
}
```

## 5. 依赖 features 选择
依赖对象可显式选择 features：
```json
{
  "dependencies": [
    {
      "name": "opencv4",
      "default-features": false,
      "features": [
        "openmp"
      ]
    }
  ]
}
```
这样能减少无关模块，降低构建成本。

## 6. 自动安装机制
使用 vcpkg toolchain 进行 configure 时，
Manifest 会自动解析并安装缺失依赖。

```bash
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"
```

流程通常是：
1. CMake 加载 toolchain。
2. 检测 `vcpkg.json`。
3. 解析依赖图与版本约束。
4. 自动安装缺失包。
5. 继续生成工程并可直接 build。

## 7. 版本锁定与可重复构建
要实现可重复构建，重点是三层：
1. `dependencies` 固定依赖集合。
2. baseline 固定默认版本集合。
3. `overrides` 固定关键包版本。

这三层共同作用，可显著降低构建波动。

## 8. KrKr2 实例分析
KrKr2 的 `vcpkg.json`（94 行）具备明显工程特征。

KrKr2 的基础元数据是 `name: krkr2` 与 `version-string: 1.5.0`。

### 8.2 平台表达式
清单中使用了：
- `linux | windows | osx`
- `android`
- `!android & !ios`
- `!windows`

这让单一清单覆盖多平台差异。

### 8.3 feature 策略
多处设置 `default-features: false`。
例如 `opencv4` 仅启用 `openmp`。

### 8.4 override 策略
KrKr2 固定 `opencv4` 为 `4.7.0#6`。
说明项目优先保障稳定与复现。

## 动手实践

### 步骤 1：创建目录
```bash
mkdir manifest_demo
cd manifest_demo
```

### 步骤 2：创建 `vcpkg.json`
```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg-tool/main/docs/vcpkg.schema.json",
  "name": "manifest-demo",
  "version-string": "0.1.0",
  "dependencies": [
    {
      "name": "fmt",
      "version>=": "10.1.0"
    },
    {
      "name": "spdlog",
      "version<": "2.0.0"
    }
  ],
  "overrides": [
    {
      "name": "fmt",
      "version": "10.2.1"
    }
  ]
}
```

### 步骤 3：创建 `vcpkg-configuration.json`
```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg-tool/main/docs/vcpkg-configuration.schema.json",
  "default-registry": {
    "kind": "builtin",
    "baseline": "b1e15efef6758eaa0beb0a8732cfa66f6a68a81d"
  }
}
```

### 步骤 4：创建构建文件
```cmake
cmake_minimum_required(VERSION 3.20)
project(manifest_demo LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
find_package(fmt CONFIG REQUIRED)
add_executable(manifest_demo main.cpp)
target_link_libraries(manifest_demo PRIVATE fmt::fmt)
```

### 步骤 5：创建程序入口
```cpp
#include <fmt/core.h>
int main() {
    fmt::print("manifest demo ok\n");
    return 0;
}
```

### 步骤 6：执行 configure 与 build
```bash
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"
cmake --build build
```

## 本节小结
- Manifest 模式优于 Classic，适合团队和 CI。
- `vcpkg.json` 管依赖声明，`vcpkg-configuration.json` 管来源策略。
- 版本约束、features、overrides 决定最终依赖结果。
- configure 阶段可自动安装依赖。
- KrKr2 配置体现了跨平台与稳定性优先实践。

## 练习题与答案

### 题目 1：模式选择
多平台长期维护项目应选哪种模式？给出两条理由。

<details>
<summary>查看答案</summary>

应选 Manifest 模式。
理由一：依赖配置文件化并纳入版本控制，团队共享清晰。
理由二：可结合 baseline 与 overrides 提高可重复构建能力。

</details>

### 题目 2：字段改写
把字符串依赖 `catch2` 改成对象依赖，要求仅非 Android 安装且关闭默认特性。

<details>
<summary>查看答案</summary>

```json
{
  "name": "catch2",
  "default-features": false,
  "platform": "!android"
}
```

</details>

### 题目 3：版本约束
写出配置：`fmt` 最低 `10.0.0`，`spdlog` 小于 `2.0.0`，并把 `fmt` 固定到 `10.2.1`。

<details>
<summary>查看答案</summary>

```json
{
  "dependencies": [
    {
      "name": "fmt",
      "version>=": "10.0.0"
    },
    {
      "name": "spdlog",
      "version<": "2.0.0"
    }
  ],
  "overrides": [
    {
      "name": "fmt",
      "version": "10.2.1"
    }
  ]
}
```

</details>

## 下一步
继续学习：[03-搜索与安装包.md](./03-搜索与安装包.md)，下一节将讲包搜索、features 查询与安装策略。
