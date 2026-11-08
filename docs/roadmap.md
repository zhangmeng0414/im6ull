# im6ull 开发路线图（裸机 → Linux → 人脸脱敏 → mTLS 上云）

- 板型：正点原子 I.MX6ULL ALPHA，**eMMC + 4.3" RGB，800x480**
- 内核：4.1.15（`~/linux/IMX6ULL/alientek_linux`），rootfs：BusyBox（`~/linux/nfs/rootfs`）
- 最终链路：OV5640 → CSI → V4L2 → 人脸检测 → **本地脱敏** → 队列 → **mTLS 双向** → 自建本地服务

## 0. 已确认输入与现状

| 项 | 结论 | 对计划的影响 |
|---|---|---|
| 屏幕 800x480 | 现有 `imx6ull-alientek-emmc.dts` 的 `&lcdif` 时序即为 800x480（35.5MHz） | **无需新建 dts 改时序**；上板后仅按实际观感微调 porch |
| 摄像头 | **暂无，待采购** | M1 摄像头步骤、M3、M4 阻塞 |
| 脱敏方式 | 本地脱敏：人脸 ROI 打码，**原图不上传、不落盘** | M4 只产出打码图（后续可加“仅特征向量”模式） |
| 云端 | 无现成服务 → **自建本地 mTLS 测试服务** | M5 拆成「服务端」与「设备端」两条并行线 |
| 联网 | USB WiFi | 内核已有 `CONFIG_RTL8188EUS=m`，已编出 `8188eu.ko`；需补 `/lib/modules` + `/lib/firmware` |

### 现状缺口（M2 必须解决）

- rootfs **没有 `/lib`**：动态链接程序（mbedTLS/ncnn/curl）无法运行，需从工具链 sysroot 拷 `libc/libm/libstdc++/ld-linux-armhf`
- rootfs **没有 `/lib/modules`**：`ov5640_camera_int.ko`、`8188eu.ko` 未部署
- rootfs **没有 `/lib/firmware`**：8188EUS 固件、OV5640 相关文件缺失（以启动日志 `request firmware failed` 为准补）
- `CMakeLists.txt` / `CMakePresets.json` / `scripts/build/imx6ull.toolchain.cmake` 仍为「待建」

## 1. 阶段计划

### M0 环境基线
打通 `编译 → 部署 → 串口日志`；落地交叉编译 CMake 工具链文件与 presets；确认 eMMC 烧录与回滚。
**DoD**：空工程 `make` 通过，`file` 显示 ARM EABI5 hard-float；一次烧录 + 回滚成功。

### M1 裸机（寄存器级，建立排障能力）
1. 启动流程 / IVT+DCD / 时钟树（CCM、PLL）
2. GPIO + UART 串口打印
3. GIC 中断 + GPT/EPIT 定时
4. I2C/SCCB 读 OV5640 ID（`0x5640`）※等模块
5. ELCDIF 点屏：纯色 → 图片 → 实时画面
6. CSI + DMA 采集一帧到 SDRAM，双缓冲上屏 ※等模块
**DoD**：串口打印摄像头 ID；屏上 320x240 实时预览；帧间隔时间戳。
无摄像头期间只做 1/2/3/5。

### M2 引导 + 内核 + rootfs（最重）
- U-Boot：eMMC 启动、`bootargs`、网络下载
- dts：以 `imx6ull-alientek-emmc.dts` 为基线（800x480 已匹配），确认触摸 IC 型号与 i2c 地址
- 内核 config：`FB_MXS`、`ov5640`、`mxc v4l2 capture`、`PXP`、`CAAM`、RTC、watchdog、`RTL8188EUS`
- rootfs 补全：`/lib`(glibc)、`/lib/modules`、`/lib/firmware`、`/etc/ssl/certs`、时区、ntp
**DoD**：断 NFS 从 eMMC 启动；`/dev/fb0` 出画面；`/dev/video0` 可抓 YUYV 帧；`wlan0` 连上 AP。

### M3 采集与显示
V4L2 mmap 采集 YUYV 640x480；**PXP 硬件 CSC + 缩放** → RGB565 → `/dev/fb0`；双缓冲、丢帧策略、帧率/CPU 统计。
**DoD**：≥15fps 预览，性能数据入库。

### M4 人脸检测 + 本地脱敏
- 框架：ncnn（ARMv7 + NEON，静态交叉编译），模型 Ultra-Light-Fast RFB-320 或 RetinaFace-MNet0.25
- baseline：OpenCV Haar 先跑通链路
- 脱敏：人脸 ROI 自适应马赛克 / 高斯 → 输出图；**原始帧仅内存处理，落盘用 tmpfs 且开机清理**
- 跟踪：IoU 连续 N 帧稳定才取证，压误检
**DoD**：50 张抽样人工核验不可识别人脸；日志证明原始帧未写入 eMMC。

### M5 mTLS 双向 + 上传（服务端 / 设备端并行）
- **服务端（可先做）**：Ubuntu 上 Python `ssl` HTTPS 服务，`ssl.CERT_REQUIRED` 强制客户端证书；测试根 CA → 服务端证书 + 设备证书；接口 `POST /v1/face-event`
- **设备端**：mbedTLS 2.28 LTS + libcurl（或 mbedTLS 手写 HTTP），TLS1.2、ECDHE、校验证书有效期与 CA
- 一机一证：私钥 `/etc/im6ull/device.key` 权限 600（进阶 CAAM Blob / 加密分区）
- 时间：RTC + NTP（时间错 → 证书校验失败）
- 队列：有界 spool、指数退避、幂等 `message_id`；证书异常**拒绝上传并告警**
**DoD**：抓包见 TLS1.2 + 客户端证书；断网 10 分钟恢复自动补传；过期/伪造证书被拒。

### M6 驻留与工程化
守护进程（init respawn / systemd）、看门狗、日志滚动、成功率指标、资源上限；可选 OTA A/B + 固件签名、HAB。
**DoD**：断电重启自恢复、掉电不丢队列、24h 无内存增长。

### M7 测试与验收
SIL（x86 mock 图像源）、集成、HIL；安全测试（错误 CA / 过期 / 时钟回拨 / 重放 / 中间人）；性能与合规文档。
**DoD**：测试报告 + 需求-用例追踪表。

### M8 文档与发布
`docs/02_architecture`（数据流）、`docs/03_interfaces`（API + JSON Schema + 证书规范）、`docs/04_release`（烧录与版本）。

## 2. 关键依赖与并行建议

| 阻塞项 | 影响范围 | 可并行推进 |
|---|---|---|
| 摄像头未到货 | M1-4/6、M3、M4 | M0、M1-1~3/5、M2 主线、M5 服务端、M4 的 x86 SIL 部分 |
| rootfs 缺 `/lib` | 所有动态链接程序 | M2 立即补 |
| 虚拟机 NAT 网络 | 板子可能访问不到 VM 上的测试服务 | 改桥接，或服务放在 Windows 主机 |
| RTC 无电池 | mTLS 校验失败 | 上电即 NTP，失败降级 |

## 3. 采购清单

1. **OV5640 摄像头模块**（正点原子官方，CSI 接口，500W）
2. **USB WiFi**：优先 **RTL8188EUS** 芯片（内核已验证可编译）；RTL8192CU 的 config 当前未开启
3. USB-TTL 串口线、5V/2A 电源（若无）

## 4. 自建 mTLS 测试服务（M5 服务端）

- 位置：Ubuntu 虚拟机；若板子走 USB WiFi，**虚拟机需桥接模式**（NAT 下板子访问不到 VM）
- 证书：`openssl` 自签测试根 CA → 服务端证书（SAN 含 VM IP）+ 每设备证书
- 服务：Python `http.server` + `ssl.SSLContext`，`verify_mode=CERT_REQUIRED`
- 先手验：`openssl s_client -connect <ip>:8443 -cert device.crt -key device.key -CAfile ca.crt`

## 5. 目录映射

| 内容 | 落位 |
|---|---|
| 裸机寄存器驱动、LCD/CSI | `platform/hal/{display,camera}`、`platform/board` |
| V4L2 / PXP 封装 | `platform/hal/camera` |
| 检测 / 脱敏 | `apps/perception/` |
| TLS / 上传 / 队列 | `middleware/comm/{tls,upload}` |
| 诊断 / 日志 | `middleware/diag`、`middleware/log` |
| 接口与证书规范 | `docs/03_interfaces` |
| HIL 用例 | `tests/hil` |

## 6. 第一批可执行动作（不等摄像头）

1. M0：落地 `scripts/build/imx6ull.toolchain.cmake`、`CMakePresets.json`、根 `CMakeLists.txt`，跑通 ARM hello
2. M2：补 rootfs 的 `/lib` + `/lib/modules` + `/lib/firmware`，把 `8188eu.ko` 部署上去并实测连接 WiFi
3. M1-1~3 + M1-5：裸机 UART → 定时器 → ELCDIF 点屏（800x480 实拍记录）
4. M5 服务端：搭 mTLS 测试服务 + 证书链，先把设备端框架（文件模拟输入）跑通

*最后更新：2026-11-05*
