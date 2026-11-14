# Phicomm S7 体脂秤 -- ESPHome 固件

基于 ESPHome 的斐讯 S7 体脂秤替代固件。

- HA 集成：ESPHome Native API 直连（不用 MQTT）
- WiFi/OTA：ESPHome 内置
- 配网：Captive Portal（无密码热点 `S7-Fallback`）
- 配置方式：改 YAML
- 自定义硬件：external_components（`components/` 下的 `s7_mcu` 与 `cs1258`）

## 前置条件

- [ESPHome](https://esphome.io/) 已安装 (2024.2.0+)
- Home Assistant 已安装并运行

## 安装

1. 复制 `secrets.yaml.example` 为 `secrets.yaml`，填入你的 WiFi：

   ```yaml
   wifi_ssid: "你的WiFi名"
   wifi_password: "你的WiFi密码"
   ```

2. 编译并刷入：

   ```bash
   cd esphome
   esphome compile phicomm-s7.yaml   # 编译
   esphome upload phicomm-s7.yaml    # 刷入（串口或OTA）
   ```

3. 在 HA 中添加 ESPHome 集成，秤会自动被发现。

## 实体列表

| 类型 | 名称 | 说明 |
|------|------|------|
| sensor | 体重 | kg |
| sensor | BMI | -- |
| sensor | 体脂率 | % |
| sensor | 水分率 | % |
| sensor | 阻抗 | ohm |
| sensor | 去脂体重 | kg |
| sensor | 基础代谢 | kcal |
| sensor | 肌肉率 | % |
| sensor | 骨量 | kg |
| sensor | WiFi RSSI | dBm |
| binary_sensor | MCU 在线 | -- |
| switch | 加热器 | 自动60秒关闭 |
| button | 测量阻抗 | 手动触发 |
| number | 年龄 | 输入框 |
| number | 身高 | cm 输入框 |
| select | 性别 | 男/女 |

## 自定义组件

本固件包含两个 ESPHome 自定义组件：

### s7_mcu -- MCU UART 体重传感器

解析秤内 MCU 通过 UART 发送的称重数据帧（C5/C6 协议）。

配置项：
- `weight_hi` / `weight_lo` -- 体重字节偏移（默认 0/1）
- `weight_div` -- 除数（默认 100）
- `weight_offset` -- 偏移量（默认 0）

### cs1258 -- CS1258 BIA 阻抗传感器

驱动 Chipsea CS1258 模拟前端，通过 GPIO bit-bang 进行 8 电极阻抗测量。

配置项：
- `pin_cs` / `pin_clk` / `pin_dio` -- GPIO 引脚（默认 16/14/12）
- `z_scale` / `z_offset` -- 阻抗校准系数

## 使用方式

1. 踩上秤 -- 体重实时更新
2. 体重稳定 3 秒 -- 自动进行阻抗扫描
3. 扫描完成 -- 体成分数据自动计算并推送到 HA
4. 下秤后 -- HA 保留最后测量结果
