# 项目名称：基于 Meshtastic 自组网的透传通信网络

本文将用于给软件开发人员介绍基于 Meshtastic 自组网的透传通信网络的实现。
从 Modbus RTU/TCP 到 Meshtastic 的接入处理，以及空中传输的 LoRa Payload 协议；
以及在中心节点中的 Meshtastic 通信，以及数据存储和处理、和对用户客户端的接入的 MQTT 接入；

## 项目介绍

整个系统将被作为一个产品推向业主用户，来为他们探察在使用用电等方面，哪些设备是存在功耗消耗大的，哪些设备能够进行远程控制，哪些设备能够进行定时控制。相当于做一次 B 超检查。

## 系统设计

整个系统分为了四个个部分：
1. 协议解析层 - 制定了版本 1 的 Modbus 基于 Meshtastic 协议以及脱离于系统免于用户接触 NodeRED 低代码变成的设备 YAML 设备解析。
2. 应用服务层 - 利用 PostgreSQL 和 MQTT 和云端用户服务通讯
3. 数据采集层 - 基于 NodeRED 搭建的 Modbus 请求原型
4. 通讯链路层 - 利用 Websocket 服务控制 Meshtastic 设备进行收发

## 系统流程

1. 首先用户在需要导入新的设备时需要通过“https://shouzhou-config-generate.seeed.cc/” 生成设备 YAML 文件，然后将 YAML 文件导入到系统中（通过 main.py --init-db 导入到数据库中）。
2. NodeRED 通过定时触发查询 PostgreSQL 数据库，然后将查询结果转换为包含 Modbus 命令的 协议 JSON，通过 Meshtastic 网络进行透传。
3. 端侧的 Meshtastic 接收并解析 JSON 出 Modbus 命令，并请求 Modbus 从站设备，然后将 得到的 Modbus 响应通过 Meshtastic 网络发送到中心设备。
4. 中心设备接收到 Modbus 响应后，解析打包为 JSON 格式，然后发送到 MQTT broker 中。

## 1. 协议解析层

### 1.1 设备 YAML 解析

设备 YAML 解析模块负责解析设备 YAML 文件，提取设备的配置信息，包括设备名称、类型、地址、功能码、寄存器地址等。解析后的配置信息将被存储到 PostgreSQL 数据库中，供后续的 Modbus 通信使用。

具体实现请看 protocol/yaml/README-YAML.md

### 1.2 传输协议解析层

协议解析层负责定义和解析基于 Meshtastic 的 Modbus 通信协议。其核心功能是将 Modbus 命令封装为紧凑型 JSON 格式，通过 Meshtastic 网络进行传输，支持设备间的透传通信。

在未来可以采用而二进制的方式进行传输。

#### 1.2.1 支持的功能

- **设备标识**：通过 `i` 字段标识发送设备。
- **设备类型**：支持 Modbus RTU (`r`) 和 Modbus TCP (`t`) 两种设备类型。
- **协议版本管理**：通过 `v` 字段标识协议版本，当前版本为 `1`。
- **通道管理**：支持通过 `p` 字段标识 RS485 端口号或通信通道。
- **Modbus 命令封装**：支持以下 Modbus 功能码：
  - 读操作：`1` (读线圈)、`2` (读离散输入)、`3` (读保持寄存器)、`4` (读输入寄存器)
  - 写操作：`5` (写单个线圈)、`6` (写单个寄存器)、`15` (写多个线圈)、`16` (写多个寄存器)
- **数据封装**：支持通过 `d` 字段传输写操作所需的数据。
- **会话管理**：可选支持 `ss` 字段，用于请求与响应的关联。

#### 1.2.2 请求报文格式

```json
{
  "i": "string",    // 发送设备的 ShortName
  "t": "string",    // 设备类型 ("r" 或 "t")
  "v": "string",    // 协议版本号
  "p": "string",    // 通道号或端口号
  "c": {            // Modbus 命令对象
    "a": int,       // 从站地址 (1-247)
    "f": int,       // 功能码 (1, 2, 3, 4, 5, 6, 15, 16)
    "r": int,       // 起始寄存器地址
    "n": int,       // 读取或写入的数量
    "d": [int]      // 写入的数据数组 (仅在功能码 ≥ 5 时有效)
  },
  "ss": int         // 会话 ID (可选)
}
```

#### 1.2.3 响应报文格式

```json
{
  "i": "string",    // 响应设备的 ShortName
  "ss": "string",   // 会话 ID (与请求一致)
  "d": [int]        // 返回的数据数组 (根据数量 `n` 的值决定长度)
}
```

#### 1.2.4 字段解释

| 字段名 | 类型   | 必填 | 说明                                                         |
| ------ | ------ | ---- | ------------------------------------------------------------ |
| `i`    | string | 是   | Meshtastic 的 ShortName，用于标识发送设备                    |
| `t`    | string | 是   | 节点接入的设备类型，支持 Modbus RTU (`r`) 和 Modbus TCP (`t`) |
| `v`    | string | 是   | 报文协议版本号，用于兼容性管理，当前为 `1`                   |
| `p`    | string | 是   | 通道号或端口号，标记 Modbus 设备接入的 RS485 端口或通信通道  |
| `c`    | object | 是   | Modbus 命令对象，包含从站地址、功能码、寄存器地址、数量和写入数据等 |
| `c.a`  | int    | 是   | 从站地址，范围 1-247                                         |
| `c.f`  | int    | 是   | 功能码，支持 1, 2, 3, 4, 5, 6, 15, 16                        |
| `c.r`  | int    | 是   | 起始寄存器地址，可为十进制或十六进制 (0x 开头)               |
| `c.n`  | int    | 是   | 读取或写入的寄存器/线圈数量                                  |
| `c.d`  | array  | 否   | 写入数据数组，仅在写操作 (功能码 ≥ 5) 时有效                 |
| `ss`   | int    | 否   | 会话 ID，用于请求和响应的关联，建议使用毫秒级时间戳          |

#### 1.2.5 通信流程

1. **请求构造**：客户端构造符合上述格式的 JSON 请求报文。
2. **发送报文**：通过 Meshtastic 网络将请求报文发送到目标设备。
3. **报文解析**：目标设备解析 JSON 并执行相应的 Modbus 操作。
4. **响应封装**：目标设备将操作结果封装为 JSON 格式的响应报文。
5. **响应返回**：通过 Meshtastic 网络将响应报文返回到客户端。

---

#### 1.2.6 示例

##### 读取保持寄存器 (功能码 3)

**请求报文**：
```json
{
  "i": "0999",
  "t": "r",
  "v": "1",
  "p": "2",
  "c": {
    "a": 1,
    "f": 3,
    "r": 40001,
    "n": 2
  }
}
```

**响应报文**：
```json
{
  "i": "000F",
  "ss": "1742891806240",
  "d": [480, 481]
}
```

##### 写入单个寄存器 (功能码 6)

**请求报文**：
```json
{
  "i": "0999",
  "t": "r",
  "v": "1",
  "p": "2",
  "c": {
    "a": 1,
    "f": 6,
    "r": 40001,
    "n": 1,
    "d": [1234]
  },
  "ss": "1742891806240",
}
```

**响应报文**：
```json
{
  "i": "000F",
  "ss": "1742891806240",
}
```

## 2. 应用服务层

应用服务层负责连接系统与云端用户服务，核心功能包括数据存储管理、消息传递和业务逻辑处理。通过 PostgreSQL 数据库和 MQTT 协议实现与云端的无缝通信。

### 2.1 功能需求

#### 2.1.1 核心功能

- **数据存储**：利用 PostgreSQL 数据库存储设备状态、设备配置、设备数据等，作为设备数据缓存。
- **消息传递**：通过 MQTT 协议实现设备与云端的双向消息通信。
- **业务逻辑处理**：处理用户请求，例如查询设备状态、控制设备操作。

### 2.2 MQTT 主题结构

消息传递设计：
设备状态消息 Topic：`livelab/device/{device_type}/{device_type}/state`
设备控制消息 Topic：`livelab/device/{device_type}/{device_id}/set`

### 2.3 设备消息示例

**设备状态消息**（发送到 MQTT Broker 中）：

```json
// livelab/device/delta_VFD_TD500_TO150G3/set
{
  "deviceId": "VFD_1",
  "data": {
    "frequency_num": 0,
    "state_check": 3
  },
  "dateTime": 1743574523898
}
```

```json
// livelab/device/Acrel_DTSD1352_C/state
{
  "deviceId": "electricity_meter_1",
  "data": {
    "AP": 0,
    "EPI": 2572,
    "CP": 0,
    "BP": 0,
    "ALLP": 0
  },
  "dateTime": 1743574523898
}
```

**设备控制消息**（从 MQTT Broker 接收）：

```json
// livelab/device/delta_VFD_TD500_TO150G3/set
{
  "data": {
    "change_state": 1
  },
  "dateTime": 1743391519602,
  "deviceId": "VFD_1"
}
```

## 3. 数据采集层(端设备)

数据采集层负责从 Modbus 从站设备采集数据，并通过 Meshtastic 网络将数据透传到中心设备。

> 在未来可以直接采用 XIAO 的方式类似与 Sensor Builder 设备的接入。

### 3.1 功能需求

#### 3.1.1 核心功能

- **指令接收**：接收处理来自中心设备的 Modbus 指令。
- **数据透传**：通过 Meshtastic 网络将数据回传到中心设备。
- **数据采集**：将解析的 Modbus 指令请求 Modbus 从机设备。

具体实现请看：`NodeRED Flow` 的 `Edge_Modbus.json`。

![edge_modbus](/assets/edge_modbus.png)

## 4. 通讯链路层 (Meshtastic 网络)

已经部署的 Meshtastic 网络是通过 XIAO + 1262 Meshtastic 网络的 Pcle 转接板接入了 R1000(CM4) 上进行的。
于是可以运行在 R1000 上的 NodeRED 或者程序可以通过 USB 串口与 XIAO 进行通信，实现 Meshtastic 网络的接入。

每一个 Meshtastic 设备都有一个唯一的 ShortName，用于标识设备，以及相同的 LoRa 频率、PSK 密钥、Meshtastic 频道配置。

```shell
meshtastic  --set-owner 'R1000_8' --set-owner-short '0008' 
meshtastic --set lora.region "US" --set lora.override_frequency 906.875
...
```

### Meshtastic网络设备组成

| 设备名称 | 作用 | 连接方式 |
|---------|------|---------|
| XIAO + 1262 | Meshtastic网络接入设备 | 通过PCIe转接板连接到R1000 |
| R1000(CM4) | 中心控制设备 | 运行NodeRED和通信程序 |
| Meshtastic节点 | 组成mesh网络的设备 | 通过LoRa无线连接 |

### 通信流程

| 步骤 | 源设备 | 目标设备 | 通信方式 | 数据格式 |
|-----|-------|---------|---------|---------|
| 1 | R1000上的应用 | XIAO | USB串口 | JSON命令 |
| 2 | XIAO | Meshtastic网络 | LoRa无线 | Meshtastic协议包 |
| 3 | 边缘Meshtastic节点 | Modbus设备 | RS485/以太网 | Modbus RTU/TCP |
| 4 | Modbus设备 | 边缘Meshtastic节点 | RS485/以太网 | Modbus响应 |
| 5 | 边缘Meshtastic节点 | XIAO | LoRa无线 | Meshtastic协议包 |
| 6 | XIAO | R1000上的应用 | USB串口 | JSON响应 |

### NodeRED 与 Meshtastic 网络的接入

由于 NodeRED 没有现成的 Meshtastic 库，所以直接使用 Python 的 Meshtastic 库然后引出 Meshtastic 接口用于 NodeRED 进行控制 Meshtastic 设备的收发。

具体实现请看：`websocket-meshtastic-bridge`。