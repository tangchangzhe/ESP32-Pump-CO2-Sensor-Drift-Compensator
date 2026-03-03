# ESP32 CO2 Drift Compensator

基于 ESP32-S3 的气泵式闭环 CO2 检测系统，内置软件漂移补偿算法解决低成本装置的漏气问题。

## 效果对比

| 原始数据 (漏气导致持续下降)                | 补偿后数据 (还原真实波动)                    |
| ------------------------------------------ | -------------------------------------------- |
| ![原始](docs/images/before_compensation.png) | ![补偿后](docs/images/after_compensation.png) |

## 项目特色

### 气泵闭环采样

采用 PWM 控制气泵进行闭环气路采样：

1. 气泵抽气循环（可调时间和力度）
2. 关泵静置等待气压平衡
3. 读取传感器数据
4. 实时上传服务器

### 自动运行模式

- 开机自动启动监测，无需手动干预
- 支持多 WiFi 热点自动切换
- 断电恢复后自动继续
- 测量时间自动对齐到 10 分钟整点

### 多点锚定漂移补偿算法

针对气泵式密闭气路装置常见的漏气问题，前端实现了创新的非线性漂移补偿：

- 用户选择多个“稳定期”作为基准锚点
- 自动计算每个锚点的 CO2 变化斜率
- 使用线性插值在锚点间平滑过渡
- 实时修正数据，还原真实 CO2 变化趋势

### 完整的 IoT 数据链路

```
ESP32 采集 → 实时HTTP上传 → PHP存储 → ECharts可视化
```

## 硬件需求

| 组件       | 型号               | 说明                 |
| ---------- | ------------------ | -------------------- |
| 主控       | ESP32-S3-DevKitC-1 | 主控制器             |
| CO2 传感器 | ZG09SR             | NDIR红外，Modbus RTU |
| 气泵       | 5V 微型气泵        | PWM 控制，闭环采样   |
| 温湿度     | DHT22/SHT30 等     | **预留接口，未使用** |

### 引脚连接

| ESP32 引脚 | 连接设备      |
| ---------- | ------------- |
| GPIO 18    | 气泵 PWM 控制 |
| GPIO 16    | ZG09SR RX     |
| GPIO 17    | ZG09SR TX     |

详细接线请参考 [硬件文档](docs/HARDWARE.md)

## 快速开始

### 1. 固件配置

```bash
cd firmware
cp src/config.h.example src/config.h
# 编辑 config.h 填入你的 WiFi 和服务器信息
```

使用 PlatformIO 编译上传：

```bash
pio run -t upload
```

### 2. 服务端部署

```bash
cd web
cp config.php.example config.php
# 编辑 config.php 填入数据库信息
```

导入示例数据（包含表结构和 1月3-9日实测数据）：

```bash
mysql -u your_user -p co2data < data/sample_data.sql
```

或手动创建空表：

```sql
CREATE TABLE co2_measurements (
  id INT AUTO_INCREMENT PRIMARY KEY,
  timestamp DATETIME NOT NULL,
  co2_ppm INT NOT NULL,
  temperature FLOAT DEFAULT NULL,  -- 预留，当前未使用
  humidity FLOAT DEFAULT NULL,     -- 预留，当前未使用
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_timestamp (timestamp)
);
```

### 3. 使用漂移补偿

1. 打开 `index.html` 网页
2. 选择时间范围查询数据
3. 在“漂移补偿”面板添加基准时间段（选择 CO2 应该稳定的时期）
4. 点击“执行漂移补偿”查看效果

## 串口指令

| 指令   | 功能                             |
| ------ | -------------------------------- |
| auto   | 恢复自动监测模式                 |
| stop   | 暂停自动，进入手动模式           |
| single | 单次测量 (手动模式下不上传)      |
| vent   | 开启通风功能                     |
| status | 查看当前状态                     |

## 配置参数

在 `config.h` 中可调整：

| 参数                  | 默认值 | 说明                              |
| --------------------- | ------ | --------------------------------- |
| PUMP_DUTY_MEASURE     | 120    | 采样时气泵 PWM 力度 (0-255)       |
| PUMP_DUTY_VENT        | 150    | 通风时气泵 PWM 力度               |
| PUMP_TIME_MEASURE     | 25     | 采样气泵运行时间 (秒)             |
| PUMP_TIME_VENT        | 50     | 通风气泵运行时间 (秒)             |
| SENSOR_STABILIZE_TIME | 20     | 气压平衡等待时间 (秒)             |
| INTERVAL_SECONDS      | 600    | 测量间隔 (固定10分钟，对齐整点)* |

> *注: 修改 `INTERVAL_SECONDS` 需同步修改时间对齐逻辑，请参考源码中 `getNextAlignedEpoch()` 函数

## 项目结构

```
esp32-co2-drift-compensator/
├── firmware/              # ESP32 固件
│   ├── src/
│   │   ├── main.cpp
│   │   └── config.h.example
│   └── platformio.ini
├── web/                   # Web 前端 + PHP 后端
│   ├── index.html         # 数据可视化页面
│   ├── get_data.php       # 数据查询接口
│   ├── sensor_data.php    # 数据接收接口
│   └── config.php.example
├── data/
│   └── sample_data.sql    # 示例数据 (1月3-9日实测)
├── docs/
│   ├── HARDWARE.md        # 硬件接线说明
│   └── images/
└── README.md
```

## 应用场景

本项目最初用于密闭空间藻类呼吸作用监测，但漂移补偿算法适用于任何存在以下问题的场景：

- 气路装置密封不良
- 需要长时间连续监测 CO2
- 传感器存在长期漂移

## 技术栈

- 固件: PlatformIO + Arduino Framework
- 传感器: ZG09SR (Modbus RTU)
- 前端: ECharts + 原生 JS
- 后端: PHP + MySQL
- 硬件: ESP32-S3 + 气泵 PWM 控制

## License

MIT License - 详见 [LICENSE](LICENSE)
