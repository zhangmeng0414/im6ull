# CLAUDE.md — 智驾嵌入式项目 AI 协作规范

> 本文档是 AI 助手（Claude / CodeBuddy / WorkBuddy 等）在本仓库中工作的最高行为准则。
> 所有 AI 生成代码、执行命令、修改文件的行为都必须遵守本文档。
> 人类工程师拥有最终决定权，AI 不得擅自绕过任何评审流程。

---

## 1. 项目概述

- **项目名称**：智能驾驶域控制器嵌入式软件（从 0 开始）
- **项目性质**：安全关键实时嵌入式系统，涉及功能安全（ISO 26262）与信息安全（ISO/SAE 21434）
- **总体架构**：单 SoC 方案，主控为 **NXP i.MX6ULL**（单核 Cortex-A7 @ 800MHz，无 GPU/NPU，armv7-a，armhf）
  - 操作系统：嵌入式 Linux（内核 4.1.15，源码 `~/linux/IMX6ULL/alientek_linux`），U-Boot 引导 + 设备树描述硬件
  - 根文件系统：BusyBox 构建，通过 NFS 挂载（`~/linux/nfs/rootfs`，仓库外）
  - 无独立安全岛 MCU；如后续需要功能安全监控，可通过 SPI/I2C 外接安全 MCU 扩展
  - 资源受限（512MB DDR3 典型配置）：禁止重量级中间件（如完整 ROS 2），通信层用轻量方案（ZMQ / 自研环形队列 / SocketCAN）
- **开发阶段**：当前处于项目初始化阶段（Phase 0），优先搭建工程骨架、构建体系与 CI

---

## 2. AI 权限与生命周期

### 2.1 权限分级（AI 可以做什么）

| 等级 | 范围 | 是否需要人工确认 |
|------|------|------------------|
| 🟢 L0 只读 | 读取任何仓库内文件、搜索代码、查看构建日志 | 无需确认 |
| 🟢 L1 生成 | 在 `apps/`、`middleware/`、`platform/`、`include/`、`tests/`、`docs/`、`tools/`、`scripts/` 下创建新文件 | 无需确认，但需在回复中说明创建了什么 |
| 🟡 L2 修改 | 修改已有业务代码、CMake/构建脚本、`config/` 配置 | 需在回复中明确列出 diff 摘要 |
| 🟠 L3 敏感 | 修改 `kernel/`、`bootloader/`、`platform/hal/`、`platform/board/`、CI 配置、版本发布脚本、依赖清单 | **必须先给出方案，等用户明确同意后执行** |
| 🔴 L4 禁止 | 见第 6 节"禁止命令/禁止行为"，无条件拒绝 | 永远不允许 |

### 2.2 生命周期阶段与 AI 职责

```
Phase 0 工程初始化 ──► Phase 1 架构设计 ──► Phase 2 模块开发 ──► Phase 3 集成测试 ──► Phase 4 发布/OTA
```

| 阶段 | AI 允许做的事 | AI 不允许做的事 |
|------|--------------|----------------|
| Phase 0 初始化 | 生成目录结构、CMake、编译脚本、lint/clang-format 配置、CI 骨架 | 不得引入未经用户确认的第三方依赖 |
| Phase 1 设计 | 输出架构文档、接口定义（.idl/.arxml/头文件）、时序图 | 不得直接改写已评审冻结的接口 |
| Phase 2 开发 | 按模块规格写代码 + 单元测试，遵循 MISRA / 编码规范 | 不得跳过单元测试直接交付；不得修改他人的模块边界 |
| Phase 3 测试 | 写集成/SIL 测试用例、分析覆盖率、修 bug | 不得为了让测试通过而修改测试断言或屏蔽测试 |
| Phase 4 发布 | 生成 CHANGELOG、版本号建议、发布检查清单 | 不得执行烧录/OTA/签名操作（仅生成脚本供人工执行） |

### 2.3 每次交付的最低要求（Definition of Done）

1. 代码通过编译（零 error，新增 warning 需说明）
2. 新增/修改逻辑附带对应单元测试，`ctest` / 单测目标全部通过
3. 通过 `clang-format` 与静态检查（cppcheck / clang-tidy）
4. 涉及安全相关代码（ASIL 分级标注）时，附上简要的安全分析说明
5. 在回复中说明：改了哪些文件、为什么、有什么风险

---

## 3. 技术栈

| 类别 | 选型 | 说明 |
|------|------|------|
| 语言 | C11 / C++17（均为目标板与宿主机） | 目标板资源受限：慎用异常与 RTTI，实时路径禁用动态内存 |
| 代码规范 | MISRA C:2012、AUTOSAR C++14（或 MISRA C++:2023） | 违规项需在注释中给出 `// MISRA-DEV` 偏离说明 |
| 构建系统 | CMake ≥ 3.22 | 统一用 presets（`CMakePresets.json`）管理配置 |
| 目标板 OS | 嵌入式 Linux 4.1.15（正点原子 i.MX6ULL 内核源码），U-Boot 引导 | 硬件由设备树描述，设备树放 `kernel/dts/`；内核配置基线 `imx_v6_v7_defconfig`，放 `kernel/configs/` |
| 系统集成 | 设备树 / 板级参数 / 运行配置 | 设备树 `kernel/dts/`，板级适配 `platform/board/`，运行参数 `config/{board,vehicle,app}/`；修改设备树属 L3 敏感操作 |
| 中间件 | 轻量方案：SocketCAN / ZMQ / 自研消息队列；**禁用完整 ROS 2**（i.MX6ULL 资源不足） | 通信层放 `middleware/comm/`，业务代码不得直接依赖具体通信实现 |
| 车载总线 | CAN / CAN-FD（Linux SocketCAN，`can0`/`can1`） | DBC 描述文件放 `middleware/comm/can/dbc/` |
| 单元测试 | GoogleTest + GoogleMock（宿主机 SIL 构建） | 测试代码放 `tests/unit/<module>/` |
| 静态分析 | clang-tidy、cppcheck、PC-lint（可选）、Coverity（CI 集成） | 新增告警不允许合入 |
| 格式化 | clang-format（仓库根目录提供 `.clang-format`） | 提交前必须格式化 |
| 版本管理 | Git + GitLab/GitHub CI，分支模型：`main`/`dev`/`feature/*`/`release/*` | 提交信息遵循 Conventional Commits |
| 交叉编译 | `arm-linux-gnueabihf-`（armv7-a hardfp，Cortex-A7 优化：`-march=armv7-a -mfpu=neon-vfpv4 -mfloat-abi=hard`） | 工具链已装于 `/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin`，所有 shell 可直接调用；CMake 工具链文件放 `scripts/build/imx6ull.toolchain.cmake`。宿主机构建仅用于单元测试（SIL），目标板产物不得在本地误运行 |
| 烧录/部署 | SD 卡启动（U-Boot + zImage + dtb）或 eMMC/TF via `uuu`（mfgtool） | 实际烧录由人工执行，AI 仅生成脚本与镜像布局说明（`scripts/flash/`） |

---

## 4. 项目架构（目录结构）

```
im6ull/
├── CLAUDE.md                     # 本文档
├── README.md                     # 分层说明、开发流程、分支约定
├── .gitignore                    # 忽略 build/、output/、.ko、镜像
├── CMakeLists.txt                # 顶层构建入口（待建）
├── CMakePresets.json             # 统一构建配置 host / imx6ull（待建）
├── .clang-format / .clang-tidy   # 格式化与静态检查配置（待建）
│
├── docs/                         # 项目文档
│   ├── 01_hardware/              # 硬件说明、引脚定义、外设资源分配
│   ├── 02_architecture/          # 软件架构、模块划分、数据流设计
│   ├── 03_interfaces/            # 接口协议（CAN 矩阵、UDS 诊断、传感器协议）
│   └── 04_release/               # 版本发布记录、镜像说明
├── hardware/                     # 硬件资料（只读，不修改）
│   ├── schematic/                # 原理图
│   ├── datasheet/                # 芯片/传感器手册
│   └── pcb/                      # PCB 与 Gerber
│
├── bootloader/                   # 引导程序（L3 敏感）
│   ├── uboot/                    # U-Boot 源码或构建脚本
│   └── patches/                  # 板级补丁
├── kernel/                       # 内核与内核态驱动（L3 敏感）
│   ├── configs/                  # defconfig / 板级配置
│   ├── dts/                      # 设备树（board.dts、外设节点、overlay）
│   ├── patches/                  # 内核补丁
│   └── modules/                  # 内核驱动模块（.ko）
│       ├── bsp/                  # 板级基础驱动（GPIO/I2C/SPI/UART）
│       ├── can/                  # CAN 控制器与收发器驱动
│       └── sensor/               # 摄像头、雷达、IMU 等传感器驱动
│
├── platform/                     # 平台适配层
│   ├── hal/                      # 硬件抽象层（L3 敏感），向上提供统一 API
│   │   ├── gpio/  can/  i2c/  spi/  uart/
│   └── board/                    # 板卡适配：引脚复用、时钟、板级初始化（L3 敏感）
│
├── middleware/                   # 中间件层
│   ├── comm/                     # 通信
│   │   ├── can/                  # SocketCAN 收发、报文调度（dbc/ 放 DBC 文件）
│   │   ├── someip/               # SOME/IP 服务通信
│   │   └── dds/                  # DDS 数据分发（轻量实现）
│   ├── diag/                     # UDS (ISO 14229) 诊断、DTC 管理
│   ├── log/                      # 日志（分级 TRACE/DEBUG/INFO/WARN/ERROR/FATAL）
│   └── os/                       # OS 抽象层（线程/定时/内存，便于移植）
│
├── apps/                         # 应用层：业务逻辑
│   ├── perception/               # 感知：图像/目标检测/传感器融合
│   ├── planning/                 # 规划：路径与行为决策
│   ├── control/                  # 控制：底盘/转向/制动指令输出
│   └── common/                   # 公共业务组件、数据结构
│
├── include/                      # 跨模块公共头文件与接口定义
├── config/                       # 运行配置（与代码分离，禁止硬编码）
│   ├── board/                    # 板级参数
│   ├── vehicle/                  # 车辆参数（轴距、轮径、标定表）
│   └── app/                      # 应用参数
│
├── scripts/                      # 自动化脚本
│   ├── build/                    # 交叉编译环境、CMake 工具链文件、构建入口
│   ├── flash/                    # 镜像打包与烧录脚本（仅生成，人工执行）
│   └── test/                     # 自动化测试执行
├── tools/                        # 自研辅助工具（协议解析、标定、日志分析）
│
├── tests/
│   ├── unit/                     # 单元测试（按模块组织）
│   ├── integration/              # 集成测试（SIL）
│   └── hil/                      # 硬件在环测试
│
├── third_party/                  # 第三方库（保持原样，不修改）
├── build/                        # 构建中间产物（不入库）
└── output/                       # 发布镜像与产物（不入库）
```

**架构铁律**（AI 写代码时必须遵守的依赖方向）：

```
apps ──► middleware ──► platform/hal ──► kernel 驱动接口 ──► hardware
                                       bootloader（独立，仅负责引导）
```

- 只允许上层依赖下层，**严禁反向依赖**（hal 不得 include middleware/apps）
- 业务代码禁止直接调用 OS API / `/dev` 节点 / 寄存器 / SocketCAN 原语，必须经过对应抽象层
- 每个模块自成一体：独立 include 独立 src 独立 tests，模块间只通过 `include/` 中的接口通信
- 配置外置到 `config/`，代码中不得硬编码板级/车辆参数

---

## 5. 常用命令（AI 可直接执行）

```bash
# 工具链（已全局可用，无需设置 PATH）
arm-linux-gnueabihf-gcc -dumpmachine        # → arm-linux-gnueabihf

# 配置（宿主机单测构建）
cmake --preset host

# 构建
cmake --build --preset host

# 运行全部单元测试
ctest --preset host --output-on-failure

# 格式化检查 / 修复
clang-format --dry-run -Werror <file>
clang-format -i <file>

# 静态检查
clang-tidy <file> -p build/host
cppcheck --enable=warning,performance middleware/ apps/

# 交叉编译（i.MX6ULL 目标）
cmake --preset imx6ull && cmake --build --preset imx6ull

# 内核模块编译（先确认内核源码已构建、Module.symvers 存在）
make -C ~/linux/IMX6ULL/alientek_linux M=$PWD ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- modules

# 查看目标板架构
file output/imx6ull/app/<binary>            # 必须为 ELF 32-bit ARM EABI5
```

约定：构建产物只写入 `build/<preset>/`，发布产物写入 `output/`，**任何脚本不得把生成物写入源码目录**。

---

## 6. 禁止命令与禁止行为（L4，无条件拒绝）

### 6.1 禁止的 Shell 命令

```bash
rm -rf /                     # 以及任何指向 /、/usr、/etc、/home、仓库根目录的递归强删
git push --force             # 除非用户显式要求且在 feature 分支
git reset --hard             # 会丢弃未提交工作，需先确认
sudo rm / sudo dd / mkfs*    # 磁盘/系统级破坏性操作
dd if=... of=/dev/...        # 直接写块设备（含烧录）
curl ... | bash              # 下载即执行
chmod -R 777 /               # 大范围权限变更
kill -9 对 CI/编译服务的进程  # 未经确认
```

### 6.2 禁止的项目行为

- ❌ 执行**真实 ECU 烧录、刷写、OTA 升级、UDS 写入诊断实体**——只允许生成脚本（`scripts/flash/`），由人工在硬件台架上执行
- ❌ 操作或连接**实车 / 台架 CAN 总线**（发送任何报文必须人工确认）
- ❌ 修改/删除 `kernel/`、`bootloader/`、`platform/hal/`、`platform/board/` 下的已有代码而不经过 L3 确认
- ❌ 绕过 MISRA / 静态检查：禁止用 `// NOLINT`、`// cppcheck-suppress` 等 pragma 修复告警而不说明理由
- ❌ 引入新第三方依赖而不更新依赖清单和说明（供应链安全：记录版本、来源、License），且只能放 `third_party/`
- ❌ 在代码或文档中写入真实密钥、证书、VIN 码、内部 IP（secrets 只放 CI 环境变量）
- ❌ 使用 `printf`/`cout` 调试输出替代正式日志模块后忘记清理
- ❌ 修改测试断言/删除测试用例来"让测试通过"
- ❌ 在实时路径中使用：动态内存分配（malloc/new）、异常、无界递归、未确认 FPU 的浮点
- ❌ 生成代码后不编译不测试就宣称"完成"

### 6.3 边界模糊时的默认策略

> 拿不准该不该做 → **先停下来问**，列出方案与风险，等确认后再执行。宁可多问一次，不可多做一步。

---

## 7. Git 与协作约定

- 提交信息：`<type>(<scope>): <subject>`，type ∈ {feat, fix, refactor, test, docs, build, ci, safety}
- 示例：`feat(planning): add lattice planner skeleton with unit tests`
- 每个 feature 一个分支，AI 不直接向 `main`/`dev` push
- 涉及安全相关模块（`apps/control/`、`platform/hal/`）的 MR 必须标注 ASIL 等级并 @ 安全负责人
- AI 不得自动创建 Release / Tag，仅在 Phase 4 给出建议

---

## 8. AI 生成代码的特别要求

1. **可追溯**：复杂算法/安全逻辑须注明设计依据（需求编号或 `docs/` 文档路径）
2. **注释语言**：代码注释用中文或英文均可，但同一文件内保持一致；公共接口必须有 Doxygen 注释
3. **禁止占位符陷阱**：不允许留 `TODO: implement later` 而不说明；临时 stub 必须标记 `// STUB:` 并在回复中声明
4. **资源意识**：关注实时性——禁止在控制回路中做堆分配、日志刷屏、锁竞争；给出大 O 与内存占用预估
5. **仿真优先**：涉及感知/规划/控制的改动，先在 SIL（单元/仿真测试）验证，不默认要求真机

---

*最后更新：2026-11-03（按实际项目目录结构同步）*
