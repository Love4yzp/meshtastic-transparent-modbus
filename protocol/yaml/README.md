```shell
python main.py --yaml config.yaml --db-user livelab --db-password livelab --db-host 192.168.101.242
python main.py --yaml config.yaml --db-user livelab --db-password livelab --db-host 120.78.123.82
python main.py --db-user livelab --db-password livelab --db-host 127.0.0.1 --init-db  --yaml YAML/20250326.yaml
```

通过将 YAML 文件导入到数据库中，可以实现设备的配置管理，包括设备名称、类型、地址、功能码、寄存器地址等。

从而能够让 NodeRED 自适应的采集 LoRaWAN、Meshtastic 的无线网络连接的 Modbus 数据。

### 数据库表结构

可以查看 main.py 中的 SQL 语句

#### 1. 设备表(`devices`)

```sql
CREATE TABLE IF NOT EXISTS devices (
    device_id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(100),
    device_type VARCHAR(50),
    system VARCHAR(50),
    location VARCHAR(50),
    communication_type VARCHAR(20) NOT NULL,
    communication_id VARCHAR(50) NOT NULL,
    connection_type VARCHAR(20) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending','ready')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_devices_status ON devices(status);
                
CREATE INDEX IF NOT EXISTS idx_devices_communication_type 
ON devices(communication_type);

CREATE INDEX IF NOT EXISTS idx_devices_connection_type 
ON devices(connection_type);
```

| 字段名 | 类型 | 说明 |
|---------------------|------------------------|-------------------------------------|
| `device_id` | VARCHAR(50) | 设备唯一标识，主键 |
| `name` | VARCHAR(100) | 设备名称 |
| `device_type` | VARCHAR(50) | 设备类型 |
| `system` | VARCHAR(50) | 系统 |
| `location` | VARCHAR(50) | 位置 |
| `communication_type` | VARCHAR(20) | 通信类型 |
| `communication_id` | VARCHAR(50) | 通信ID |
| `connection_type` | VARCHAR(20) | 连接类型 |
| `status` | VARCHAR(20) | 状态 |
| `created_at` | TIMESTAMP WITH TIME ZONE | 创建时间 |
| `updated_at` | TIMESTAMP WITH TIME ZONE | 更新时间 |

#### 2. 设备连接参数表(`device_connection_params`)

```sql
CREATE TABLE IF NOT EXISTS device_connection_params (
    device_id VARCHAR(50) PRIMARY KEY 
    REFERENCES devices(device_id) ON DELETE CASCADE,
    port_number VARCHAR(10) NOT NULL,
    slave_address VARCHAR(10) NOT NULL,
    baud_rate VARCHAR(10) DEFAULT '9600',
    stop_bits INTEGER DEFAULT 1,
    data_bits INTEGER DEFAULT 8,
    parity VARCHAR(10) DEFAULT 'None',
    timeout FLOAT DEFAULT 3.0,
    retry_count INTEGER DEFAULT 3,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

| 字段名 | 类型 | 说明 |
|---------------------|------------------------|-------------------------------------|
| `device_id` | VARCHAR(50) | 设备唯一标识，主键 |
| `port_number` | VARCHAR(10) | 端口号，NOT NULL |
| `slave_address` | VARCHAR(10) | 从机地址，NOT NULL |
| `baud_rate` | VARCHAR(10) | 波特率，默认'9600' |
| `stop_bits` | INTEGER | 停止位，默认1 |
| `data_bits` | INTEGER | 数据位，默认8 |
| `parity` | VARCHAR(10) | 校验位，默认'None' |
| `timeout` | FLOAT | 超时时间，默认3.0 |
| `retry_count` | INTEGER | 重试次数，默认3 |
| `created_at` | TIMESTAMP WITH TIME ZONE | 创建时间 |
| `updated_at` | TIMESTAMP WITH TIME ZONE | 更新时间 |

#### 3. 设备属性表(`device_properties`)

这个是最主要的。

```sql
CREATE TABLE IF NOT EXISTS device_properties (
    property_id VARCHAR(50) PRIMARY KEY,
    device_id VARCHAR(50) 
    REFERENCES devices(device_id) ON DELETE CASCADE,
    request_type VARCHAR(20) NOT NULL,
    register_address VARCHAR(10) NOT NULL,
    function_code VARCHAR(10) NOT NULL,
    data_length INTEGER NOT NULL,
    byte_order VARCHAR(20) DEFAULT 'big_endian',
    word_order VARCHAR(20) DEFAULT 'big_endian',
    scale_factor FLOAT DEFAULT 1.0,
    value_offset FLOAT DEFAULT 0.0,
    data_type VARCHAR(20) NOT NULL,
    bit_switches JSONB DEFAULT '[]',
    read_values JSONB DEFAULT '[]',
    enum_values JSONB DEFAULT '[]',
    last_command_value INTEGER DEFAULT 0,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending','ready')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

| 字段名 | 类型 | 说明 | 
|---|---|---| 
| `id` | SERIAL | 自增主键 | 
| `device_id` | VARCHAR(50) | 设备ID，外键关联devices表 | 
| `property_id` | VARCHAR(50) | 属性ID，NOT NULL | 
| `request_type` | VARCHAR(20) | 请求类型，NOT NULL | 
| `description` | TEXT | 属性描述 | 
| `function_code` | VARCHAR(10) | 功能码，NOT NULL | 
| `register_address` | VARCHAR(10) | 寄存器地址，NOT NULL | 
| `data_length` | INTEGER | 数据长度，NOT NULL |
| `byte_order` | VARCHAR(20) | 字节序，默认'big_endian' |
| `word_order` | VARCHAR(20) | 字序，默认'big_endian' |
| `scale_factor` | FLOAT | 比例因子，默认1.0 |
| `value_offset` | FLOAT | 值偏移，默认0.0 |
| `data_type` | VARCHAR(20) | 数据类型，NOT NULL |
| `bit_switches` | JSONB | 位开关配置 |
| `read_values` | JSONB | 读取值配置 |
| `enum_values` | JSONB | 枚举值配置 |
| `last_command_value`| INTEGER | 上次命令值 |
| `status` | VARCHAR(20) | 属性状态，默认'pending' |
| `session_id` | VARCHAR(50) | 会话ID | 
| `retry_count` | INTEGER | 重试次数，默认0 |
| `created_at` | TIMESTAMP WITH TIME ZONE| 创建时间，默认NOW() | 
| `updated_at` | TIMESTAMP WITH TIME ZONE| 更新时间，默认NOW() |

其中 `scale_factor`、`value_offset` 就是比例因子和值偏移， Y=scale_factor * X + value_offset（AX+B）。


### 内容示例

#### read_values

```js
let _value;

switch (fc_length) {
    case 1:
        // 8位数据（1字节）
        _value = original_value[0];
        break;
        
    case 2:
        // 16位数据（2字节）
        _value = (original_value[0] << 8) | original_value[1];
        break;
        
    case 4:
        // 32位数据（4字节）
        _value = (original_value[0] << 24) | 
                (original_value[1] << 16) | 
                (original_value[2] << 8) | 
                original_value[3];
        break;
        
    default:
        node.error(`Unexpected data length: ${data_length}`);
        break;
}
// 创建或更新 property_id: value 格式
const updatedEnumValues = {
    [property.property_id]: _value
};

// 生成更新 SQL
msg.query = `
    UPDATE device_properties
    SET 
        read_values = $1::jsonb,
        status = 'success',
        updated_at = CURRENT_TIMESTAMP
    WHERE 
        id = $2;
`;

msg.params = [JSON.stringify(updatedEnumValues), property.id];
```

read_values 的作用就是把每一个设备的 id，和读取到的值，打包成一个 JSON 格式，然后更新到数据库中。类似如下：

```json
{
    "Temperature": 25.0,
    "Humidity": 50.0
}
```
