# Phicomm S7 Body Fat Scale — ESP8266 替代固件

斐讯 S7 体脂秤 ESP8266（CSM64F02）替代固件。斐讯云服务已关闭，此固件让你的 S7 体脂秤重获新生，原生接入 Home Assistant。

仓库提供两套方案：

| 方案 | 接入方式 | 状态 |
|------|----------|------|
| **ESPHome 方案（推荐）** | ESPHome Native API 直连 HA，多成员测量 + 待确认归档 | 当前主线，持续维护 |
| 原有 PlatformIO / Arduino 固件 | MQTT Discovery | 遗留保留，见文末 |

## ESPHome 方案

### 组成

- `esphome/phicomm-s7.yaml` — ESPHome 设备配置入口（ESP8266），含 Native API、Web 配置界面、OTA
- `esphome/components/` — 自定义组件：S7 MCU UART 协议（`s7_mcu`）与 CS1258 BIA AFE（`cs1258`）
- `homeassistant/packages/phicomm_s7.yaml` — HA 后端逻辑包：成员资料、待确认队列、归档脚本与自动化、模板传感器
- `homeassistant/lovelace/phicomm_s7_cards.yaml` — Lovelace 面板卡片（体脂秤状态 + 4 张成员卡）

### 功能

- **体重测量** — MCU UART 协议解析，实时体重读数；体重稳定后自动触发 CS1258 阻抗扫描
- **8 电极阻抗测量** — CS1258 BIA AFE，9 通道扫描（双脚电极 + 双手握柄）
- **体成分分析** — 体脂率、水分率、BMI、去脂体重、基础代谢、肌肉率、骨量
- **4 名家庭成员** — 每人独立档案（姓名 / 出生日期 / 身高 / 性别），各自直接发起测量，互不干扰
- **待确认归档队列** — 测量记录先进 5 槽队列，面板按体重相似度给出归属建议，确认后归入成员，未确认可舍弃；配合设备端 ACK 机制保证记录不丢失
- **HA 侧参数配置** — 年龄、身高、性别在 HA 面板中直接修改，自动同步到秤
- **加热器控制** — GPIO13 脚垫加热，HA 开关实体（带超时保护）
- **中文 Web UI** — 秤机网页配置（WiFi / MQTT / 用户资料 / 高级调试，折叠显示）
- **OTA 升级** — ESPHome OTA

> 说明：设备端 `set_active_member` 只接受成员号 0–3，第 4 名成员的档案同样会同步到秤，测量记录归属由 HA 归档时指定，功能上与其他成员一致。

### 安装步骤

1. **刷入固件**：用 ESPHome 编译 `esphome/phicomm-s7.yaml`（需将 `esphome/components/` 一并放入 ESPHome 配置目录），首次串口刷入，之后走 OTA。秤重启后创建热点 `S7-xxxxxx`，连接后访问 `192.168.4.1` 配置 WiFi。
2. **接入 HA**：HA → 设置 → 设备与服务 → 添加集成 → **ESPHome**，填秤的 IP。设备名保持默认 `phicomm-s7`（下述配置按此实体 ID 编写）。
3. **安装 HA 后端包**：把 `homeassistant/packages/phicomm_s7.yaml` 复制到 HA 的 `/config/packages/`，在 `configuration.yaml` 中加入：

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

   开发者工具 → 检查配置通过后，**重启 HA**（packages 需重启生效）。

4. **安装前端卡片依赖**：HACS → 前端，安装 **Mushroom**（`piitaya/lovelace-mushroom`）和 **Stack In Card**（`custom-cards/stack-in-card`）；搜索不到时用"自定义存储库"按仓库地址添加。
5. **添加面板**：仪表盘 → 编辑 → 添加卡片 → 手动（Manual），把 `homeassistant/lovelace/phicomm_s7_cards.yaml` 的完整内容粘贴**一次**保存，强刷浏览器（Ctrl+Shift+R）。

### 使用流程

1. 点成员卡头部编辑资料（姓名 / 出生日期 / 身高 / 性别）
2. 调用 `script.s7_measure_memberN`（N = 1–4）发起测量 → 状态变"等待上秤"→ 站上秤唤醒设备
3. 测量完成后记录进入待确认队列，面板自动给出归属建议（按体重相似度匹配）
4. 点对应成员按钮归档，或点"舍弃"丢弃；归档后各成员卡显示该成员最近一次体成分数据

固件内的 "HA API ACK Required" 开关请保持打开，启用"入队后再确认"的可靠传输逻辑。

## 硬件

### 斐讯 S7 硬件架构

```
┌─────────────────────────────────────┐
│  Phicomm S7 Body Fat Scale          │
│                                     │
│  ┌──────────┐    UART     ┌──────┐  │
│  │  MCU     │◄──────────►│ESP8266│  │
│  │(称重ADC) │  115200     │CSM64F│  │
│  │          │  C5/C6 协议 │ 02   │  │
│  └──────────┘             └──┬───┘  │
│                              │GPIO  │
│  ┌──────────┐   bit-bang   │      │
│  │  CS1258  │◄────────────┘      │
│  │(BIA AFE) │  CS=16,CLK=14      │
│  │ 8电极    │  DIO=12            │
│  └──────────┘                     │
│                                     │
│  加热器 ──── GPIO13 (高电平有效)     │
│  4脚垫电极 + 4握柄电极 = 8电极      │
└─────────────────────────────────────┘
```

- **ESP8266** (CSM64F02) — WiFi + ESPHome 固件载体
- **MCU** — 称重传感器 ADC，通过 UART 发送体重数据
- **CS1258** — Chipsea BIA 模拟前端，8 电极阻抗测量
- **加热器** — GPIO13，脚垫加热

## 体成分算法

使用 [bodymiscale](https://github.com/dckiller51/bodymiscale) / Xiaomi Zepp Life 验证公式：

- **LBM (去脂体重)**: 基于身高、体重、阻抗、年龄的多元回归
- **体脂率**: Xiaomi / Zepp Life 公式，区分男女
- **水分率**: `(100 - 体脂率) × 0.7` + 修正
- **BMR (基础代谢)**: Katch-McArdle 公式: `370 + 21.6 × LBM`
- **骨量**: bodymiscale 公式
- **肌肉率**: `体重 - 脂肪质量 - 骨量`

## MCU UART 协议

秤的 MCU 通过 UART 发送称重数据，帧格式：

```
C5 <len> <cmd> <payload...> <xor>
```

- `cmd = 0x10`：体重数据帧
- 默认 `payload[0]:payload[1]` 为体重高低字节，÷ 100 = kg
- 可在 Web UI 的「高级设置」中调整字节偏移和除数

### CS1258 通道映射

| 通道 | 代码 | 路径 |
|------|------|------|
| 0xEB | lrLeg | 左脚→右脚 |
| 0xBE | rlLeg | 右脚→左脚 |
| 0x14 | lrHand | 左手→右手 |
| 0xDB | rLeg | 右脚 |
| 0x8E | lLeg | 左脚 |
| 0x3C | lhLl | 左手→左脚 |
| 0x28 | lhRl | 左手→右脚 |
| 0x7D | rhLl | 右手→左脚 |
| 0x69 | rhRl | 右手→右脚 |

## FAQ

**Q: 阻抗读数为 0？**
A: 确保双脚踩在金属电极上、双手握住握柄。CS1258 需要人体导通才能测量。

**Q: HA 面板报"自定义卡片不存在"？**
A: 未安装 Mushroom 或 Stack In Card，按上文 HACS 步骤安装后强刷浏览器。

**Q: 仪表盘出现多组重复面板？**
A: 手动卡片被粘贴了多次。编辑仪表盘删除多余组，只保留一份；更新卡片时应"编辑现有卡片替换内容"，不要重复添加。

**Q: HA 检查配置提示实体不存在？**
A: 确认已通过 ESPHome 集成接入设备，且设备 name 为默认的 `phicomm-s7`。

**Q: MCU 离线？**
A: MCU 在无操作约 30 秒后进入休眠，踩上秤即可唤醒。

**Q: 加热器不工作？**
A: 在 Web UI 中将加热器 GPIO 设为 `13`。默认公版固件加热器是禁用的 (`-1`)。

## 遗留方案：PlatformIO / Arduino 固件（MQTT）

原 Arduino 固件（`src/`、`platformio.ini`、`release/`）仍保留，通过 MQTT Discovery 接入 HA：

- 编译刷入：

  ```bash
  git clone https://github.com/Praepes/phicomm-s7-ha.git
  cd phicomm-s7-ha
  pio run -e s7_esp8266_1m          # 编译
  pio run -e s7_esp8266_1m -t upload # 串口刷入
  ```

- HA 需配置 [MQTT 集成](https://www.home-assistant.io/integrations/mqtt/)，秤连接后自动 Discovery 以下实体：体重 / BMI / 体脂率 / 水分率 / 阻抗 / 去脂体重 / 基础代谢 / 肌肉率 / 骨量 / WiFi RSSI / MCU 在线 / CS1258 / 加热器开关 / 测量阻抗按钮 / 年龄 / 身高 / 性别
- MQTT Topic：`s7/phicomm_s7/state`（状态 JSON）、`s7/phicomm_s7/availability`、`s7/phicomm_s7/cmd/{impedance,heater,age,height,sex}`
- OTA：编辑 `platformio.ini` 的 `upload_port` 后 `pio run -e s7_esp8266_1m_ota -t upload`，或访问 `http://<秤IP>/update`

## 原项目与致谢

本项目的硬件逆向、协议研究及早期替代固件工作承接自原开源项目 [Praepes/phicomm-s7-ha](https://github.com/Praepes/phicomm-s7-ha)。在此特别感谢原作者 **Praepes** 的先行探索、代码贡献与公开分享，为后续 ESPHome 适配和 Home Assistant 扩展提供了重要基础。

同时感谢 [bodymiscale](https://github.com/dckiller51/bodymiscale) 等相关开源项目及社区贡献者提供的算法参考和技术资料。本仓库后续修改由本仓库维护者独立整理与维护；如引用或再分发，请遵循各上游项目的许可证及署名要求。

## License

MIT — 详见 [LICENSE](LICENSE)
