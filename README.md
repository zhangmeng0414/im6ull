# IM6ULL 智驾嵌入式项目

基于 NXP i.MX6ULL (ARM Cortex-A7) 的智能驾驶域控制器软件开发工程。
采用「Bootloader → Kernel/BSP → Middleware → Application」分层结构，
面向车规级模块化、可测试、可裁剪的工程组织方式。

## 目录结构

```
im6ull/
├── docs/                 # 项目文档
│   ├── 01_hardware/      # 硬件说明、引脚定义、外设资源分配
│   ├── 02_architecture/  # 软件架构、模块划分、数据流设计
│   ├── 03_interfaces/    # 接口协议（CAN 矩阵、UDS 诊断、传感器协议）
│   └── 04_release/       # 版本发布记录、镜像说明
├── hardware/             # 硬件资料（从设计端同步，只读）
│   ├── schematic/        # 原理图
│   ├── datasheet/        # 芯片/传感器手册
│   └── pcb/              # PCB 文件与 Gerber
├── bootloader/           # 引导程序
│   ├── uboot/            # U-Boot 源码（或指向外部源码的构建脚本）
│   └── patches/          # 对 U-Boot 的板级补丁
├── kernel/               # 内核与内核态驱动
│   ├── configs/          # defconfig / 板级配置文件
│   ├── dts/              # 设备树（board.dts、外设节点）
│   ├── patches/          # 内核补丁
│   └── modules/          # 内核驱动模块（.ko）
│       ├── bsp/          # 板级基础驱动（GPIO/I2C/SPI/UART）
│       ├── can/          # CAN 控制器与收发器驱动
│       └── sensor/       # 摄像头、雷达、IMU 等传感器驱动
├── platform/             # 平台适配层（HAL + 板级配置）
│   ├── hal/              # 硬件抽象层，向上提供统一接口
│   │   ├── gpio/  can/  i2c/  spi/  uart/
│   └── board/            # 具体板卡适配、引脚复用、时钟配置
├── middleware/           # 中间件层
│   ├── comm/             # 通信
│   │   ├── can/          # CAN/CAN-FD 收发、报文调度
│   │   ├── someip/       # SOME/IP 服务通信
│   │   └── dds/          # DDS 数据分发
│   ├── diag/             # UDS (ISO 14229) 诊断、DTC 管理
│   ├── log/              # 日志与运行记录
│   └── os/               # 操作系统适配（线程/定时/内存）
├── apps/                 # 应用层
│   ├── perception/       # 感知：图像/目标检测/传感器融合
│   ├── planning/         # 规划：路径与行为决策
│   ├── control/          # 控制：底盘/转向/制动指令输出
│   └── common/           # 公共业务组件、数据结构
├── include/              # 跨模块公共头文件与接口定义
├── config/               # 运行配置（与代码分离）
│   ├── board/            # 板级参数
│   ├── vehicle/          # 车辆参数（轴距、轮径、标定表）
│   └── app/              # 应用参数
├── scripts/              # 自动化脚本
│   ├── build/            # 构建、交叉编译环境
│   ├── flash/            # 镜像打包与烧录
│   └── test/             # 自动化测试执行
├── tools/                # 自研辅助工具（协议解析、标定、日志分析）
├── tests/                # 测试
│   ├── unit/             # 单元测试
│   ├── integration/      # 集成测试
│   └── hil/              # 硬件在环测试
├── third_party/          # 第三方库（保持原样，不修改）
├── build/                # 构建中间产物（不入库）
└── output/               # 发布镜像与产物（不入库）
```

## 分层约定

| 层级 | 职责 | 依赖方向 |
|---|---|---|
| apps | 业务逻辑（感知/规划/控制） | 依赖 middleware、include |
| middleware | 通信、诊断、日志、OS 适配 | 依赖 platform、include |
| platform | HAL 与板级适配，屏蔽硬件差异 | 依赖 kernel 驱动接口 |
| kernel / bootloader | 内核驱动、设备树、引导 | 直接操作硬件 |

**规则：上层只能调用下层，禁止反向依赖；跨层通信走 include 中定义的接口。**

## 开发流程

1. 硬件资料归档到 `hardware/` 与 `docs/01_hardware/`
2. 设备树与驱动在 `kernel/` 开发，模块单独编译为 `.ko`
3. HAL 在 `platform/hal/` 提供统一 API，业务代码不直接访问 `/dev` 节点
4. 应用在 `apps/` 开发，配置放 `config/`，禁止硬编码
5. 构建与烧录统一走 `scripts/`

## 环境说明

- 交叉编译器：`arm-linux-gnueabihf-gcc`（Linaro GCC 4.9.4）
  路径：`/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin`
  已配置 `/etc/profile.d/arm-gcc.sh` 与 `/usr/local/bin` 软链，登录与非登录 shell 均可直接调用
- 目标板：正点原子 i.MX6ULL
  内核源码：`~/linux/IMX6ULL/alientek_linux`；U-Boot 源码：`~/linux/IMX6ULL/uboot`
- 根文件系统通过 NFS 挂载：`~/linux/nfs/rootfs`

## 分支与提交约定

- `main`：可发布分支，`dev`：日常开发，`feature/*`：功能分支
- 提交信息格式：`<模块>: <变更说明>`，例如 `kernel/can: 修复 CAN 报文丢失`
- 生成的 `.ko`、镜像、构建目录不提交

## 开发环境与调试通道（已固化）

### 网络拓扑

```
板子 eth0 192.168.2.50 ──网线──► 主机 2.5G 网口 192.168.2.1
                                   │ VMware 桥接
                                   ▼
                    虚拟机 ens33 192.168.2.10（TFTP + NFS 服务器）
```

| 角色 | 地址 | 说明 |
|---|---|---|
| 开发板 | 192.168.2.50 | 静态 IP，U-Boot 与内核启动参数一致 |
| 主机网口 | 192.168.2.1 | 板子直连，VMware 以桥接方式接入 |
| 虚拟机 | 192.168.2.10 | tftpd-hpa(69) + nfs-kernel-server(2049) |

> 注意：主机 Realtek 网卡的 `PowerSavingMode`、`EnableGreenEthernet`、`GigaLite`、`SelectiveSuspend` 必须关闭，
> 否则空闲时会断开链路（表现为网卡 `Disconnected / 0 bps`，TFTP 直接失败）。
> 链路速率为 100 Mbps（板载 FEC 为百兆 PHY）。

### U-Boot 固化环境（已 saveenv 写入 eMMC）

```
ipaddr=192.168.2.50
netmask=255.255.255.0
serverip=192.168.2.10
gatewayip=192.168.2.1
bootdelay=3
bootcmd=ping 192.168.2.10; tftp 0x80800000 zImage; tftp 0x83000000 imx6ull-alientek-emmc.dtb; bootz 0x80800000 - 0x83000000
bootargs=console=ttymxc0,115200 root=/dev/nfs rw ip=192.168.2.50:192.168.2.10:192.168.2.1:255.255.255.0::eth0:off nfsroot=192.168.2.10:/home/ubuntu/linux/nfs/rootfs,v3,tcp
```

复位后自动完成「TFTP 取内核 → 取设备树 → 启动 → NFS 挂根文件系统」。
需要进入 U-Boot 命令行时，在串口上电 3 秒内按 `Ctrl+C`。
`bootcmd` 首条 `ping` 用于预热 ARP，避免偶发的 `TFTP error: 'no ARP reply'`。

### 串口（仅作救砖与早期观察）

- Windows 侧设备：`COM3`（CH340），`115200 8N1`，无流控
- 虚拟机内：`/dev/ttyUSB0`（ch341-uart）
- 注意：串口为独占资源，MobaXterm 的 Serial 会话打开时，脚本无法同时占用；
  虚拟机 USB 直通在 UHCI 下偶发 `-110` 超时，必要时执行 `usbreset` 或重插

### SSH 远程登录

```bash
ssh -i C:\Users\Administrator\.ssh\id_ed25519 root@192.168.2.50
```

- 服务端：dropbear 2022.83，动态链接（`/usr/sbin/dropbear`、`dropbearkey`）
- 自启：`/etc/init.d/rcS` 末尾 `/usr/sbin/dropbear -R -p 22`
- 主机密钥：`/etc/dropbear/dropbear_rsa_host_key`、`dropbear_ecdsa_host_key`
- 公钥登录：`/root/.ssh/authorized_keys`（属主 `root:root`，目录 700 / 文件 600）

### rootfs 关键依赖（缺一不可）

| 项 | 位置 | 作用 |
|---|---|---|
| glibc 运行时 | `/lib`（72 个 .so，含 `ld-linux-armhf.so.3`） | 动态程序启动 |
| NSS 模块 | `/lib/libnss_files.so.2`、`libnss_dns.so.2` | `getpwnam`、域名解析 |
| nsswitch | `/etc/nsswitch.conf` | `passwd: files`、`hosts: files dns` |
| 账号库 | `/etc/passwd`、`/etc/group` | 无此文件时 dropbear 报 `nonexistent user` |
| libcrypt / libutil | `/lib/libcrypt.so.1`、`libutil.so.1` | dropbear 运行依赖 |
| 设备节点 | devtmpfs 挂载 + `mknod /dev/console c 5 1` | BusyBox init 控制台 |

> 踩坑记录：静态链接的 glibc 程序在板子上 `getpwnam("root")` 返回 `NOT FOUND`，
> 即使补齐 `libnss_files.so.2` 仍无效。dropbear 因此改为**动态链接**编译。

## 已知限制

- 板子**无 RTC 电池**，系统时间固定从 `1970-01-01` 起算；做 TLS/mTLS 之前必须先接入 NTP
- `/lib/modules`、`/lib/firmware` 目前为空，OV5640（`ov5640_camera_int.ko`）与
  USB WiFi（`8188eu.ko`）模块尚未部署
- 摄像头尚未采购；4.3 寸屏型号为 `ATK-LCD-4.3-800x480`，设备树时序与之一致，无需改屏参

## CI/CD 平台（Jenkins + JFrog Artifactory）

### 拓扑

```
GitHub  zhangmeng0414/im6ull
   │  SCM 轮询 */5 * * * *（GitHub 无法回调内网，不用 webhook）
   ▼
Jenkins 2.580.2      192.168.2.10:8080      JDK 21
   │  打包后 curl PUT
   ▼
JFrog Artifactory    192.168.2.10:8082      OSS 7.161.15
   │  JDBC
   ▼
PostgreSQL 12        127.0.0.1:5432
```

### 访问信息

| 服务 | 地址 | 账号 | 密码 |
|---|---|---|---|
| Jenkins | http://192.168.2.10:8080 | admin | im6ull2026 |
| Artifactory | http://192.168.2.10:8082 | admin | password |
| 产物浏览 | http://192.168.2.10:8082/artifactory/example-repo-local/im6ull/ | admin | password |

### 安装形态（全部用户级，不依赖系统包）

| 组件 | 路径 |
|---|---|
| Jenkins war | `/home/ubuntu/tools/jenkins.war`（2.580.2） |
| JDK 21 | `/home/ubuntu/tools/jdk-21.0.13+9` |
| Jenkins HOME | `/home/ubuntu/.jenkins` |
| Artifactory | `/home/ubuntu/tools/artifactory-oss-7.161.15` |
| PostgreSQL 12 | `/home/ubuntu/tools/pg/usr/lib/postgresql/12/bin` |
| PG 数据目录 | `/home/ubuntu/pgdata` |

### systemd 开机自启

| Unit | 端口 | 说明 |
|---|---|---|
| `im6ull-pg.service` | 5432 | Type=forking，数据目录 `~/pgdata` |
| `im6ull-jenkins.service` | 8080 | Type=simple，Restart=always |
| `im6ull-artifactory.service` | 8081/8082 | Requires `im6ull-pg`，启动约 1–2 分钟 |

```bash
sudo systemctl status im6ull-pg im6ull-jenkins im6ull-artifactory
sudo journalctl -u im6ull-jenkins -f
sudo systemctl restart im6ull-artifactory
```

### Jenkins 任务 `im6ull-build`

| 配置项 | 值 |
|---|---|
| 类型 | Pipeline（Pipeline script from SCM） |
| 仓库 | `git@github.com:zhangmeng0414/im6ull.git` |
| 分支 | `main` |
| 脚本路径 | `Jenkinsfile` |
| 触发器 | SCM 轮询 `*/5 * * * *` |
| 构建保留 | 最近 30 次 |

凭据（Manage Jenkins → Credentials → System → Global）：

| ID | 类型 | 内容 |
|---|---|---|
| `github-deploy-key` | SSH Username with private key | `git` + `/home/ubuntu/.ssh/id_ed25519` |
| `artifactory-admin` | Username with password | `admin` / `password` |

### Jenkinsfile 流水线

`Checkout` → `Inspect` → `Package` → `Upload to JFrog`

- 版本号：`v$(date +%Y%m%d)-$(git rev-parse --short HEAD)`
- 打包：`git archive` 生成 `im6ull-<ver>.tar.gz` 与 `manifest.txt`（version / commit / branch / built / jenkins build# / sha256）
- 上传：`curl -T` 到 `example-repo-local/im6ull/<ver>/`
- 归档：`archiveArtifacts output/**`，收尾 `deleteDir()`

### Artifactory 配置

| 项 | 值 |
|---|---|
| 数据库 | PostgreSQL 12 |
| JDBC | `jdbc:postgresql://127.0.0.1:5432/artifactory` |
| DB 用户 | `artifactory` / `artifactory` |
| 配置文件 | `~/tools/artifactory-oss-7.161.15/var/etc/system.yaml` |
| 仓库 | 内置 `example-repo-local`（OSS 版不支持 API 建仓） |
| JVM | `-Xms512m -Xmx2g` |

### 常用命令

```bash
# 手动触发构建
java -jar ~/tools/jenkins-cli.jar -s http://192.168.2.10:8080/ \
  -auth admin:im6ull2026 build im6ull-build -s -v

# 查看产物清单
curl -u admin:password \
  http://192.168.2.10:8082/artifactory/api/storage/example-repo-local/im6ull

# 直连数据库
~/tools/pg/usr/lib/postgresql/12/bin/psql -h 127.0.0.1 -p 5432 -U postgres
```

### 已知限制

- Artifactory OSS 免费版**不能**通过 REST API 建仓库，需在 8082 页面手动创建专用仓库（如 `im6ull-local`）
- Artifactory 重启后约 1–2 分钟才返回 200，期间 8082 会 502，属正常初始化
- Jenkins 插件走官方源，下载慢；紧急时可手工下载 `.jpi` 放入 `~/.jenkins/plugins`
- 管理员密码固定为 `im6ull2026`，由 `~/.jenkins/init.groovy.d/01-security.groovy` 在首次启动创建；初始随机密码文件已清理
