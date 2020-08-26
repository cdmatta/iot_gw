### 1. 功能描述

**已支持以下功能**：

- 数据管理模块(使用 sqlite3)

- 支持与服务器通过MQTT协议通信，保持长连接
- 支持解析JSON格式的终端数据

- 支持从文件读取终端数据

- 离线时缓存来自终端的数据

- 在线时发布离线时缓存的数据

- 使用同一个TCP连接发送数据

- 从文件读取配置信息，例如数据库路径、服务器地址、id等信息

- 支持数据模板，根据数据模板重新格式化来自终端的数据、添加自定义属性、预定义属性(例如添加时间戳)等，从而生成新的JSON数据
- 支持从串口读取终端的数据

**计划开发的功能**：

- 支持缓存来自终端的任意类型数据
- 支持TSL加密
- 提供配置信息的WEB界面

### 2. 使用示例

在本地启动 MQTT broker，监听 1883 端口，例如使用 mosquitto：

```
mosquitto -v -p 1883
```

或者修改配置文件 `gw.toml`，指定可用的 MQTT broker：

```
[server]
address = "127.0.0.1:1883"
```

#### （1） 从文件读取数据（默认）

修改 Cargo.toml 文件，开启 `data_interface_text_file` 特性（默认开启此特性）：

```
[features]
#default = ["data_interface_serial_port"]
default = ["data_interface_text_file"]
data_interface_serial_port = []
data_interface_text_file = []
```

修改配置文件（默认是 gw.toml），指定串口，并且将数据接口类型设置为 `text_file`（默认为此配置）：

```
[data_if]
#if_name = "/dev/ttyS14"
#if_type = "serial_port"
if_name = "./data_if.txt"
if_type = "text_file"
```

编译运行网关程序：

```
cargo run -- -c gw.toml
```

在文件中写入数据（需要有换行 `'\n'`）：

```
echo "{\"id\":1,\"name\":\"SN-001\",\"temperature\": 27.45,\"humidity\": 25.36,\"voltage\": 3.88,\"status\": 0}" > data_if.txt
```

**说明**：网关程序每隔 1 秒读清一次文件，读清后需要手动写入数据。

#### (2) 从串口读取数据

修改 Cargo.toml 文件，开启 `data_interface_serial_port` 特性

```
[features]
default = ["data_interface_serial_port"]
#default = ["data_interface_text_file"]
data_interface_serial_port = []
data_interface_text_file = []
```

修改配置文件（默认是 gw.toml），指定串口，并且将数据接口类型设置为 `serial_port`：

```
[data_if]
if_name = "/dev/ttyS14"
if_type = "serial_port"
#if_name = "./data_if.txt"
#if_type = "text_file"
```

编译运行网关程序：

```
cargo run -- -c gw.toml
```

使用外部设备按 HDTP 的帧格式（见 HDTP [README](/hdtp/README.md)）向串口发送数据（波特率 115200）：

```
{"id":1,"name":"SN-001","temperature": 27.45,"humidity": 25.36,"voltage": 3.88,"status": 0}
```

Arduino 示例程序：

```
void setup() {
  Serial.begin(115200);
}

uint16_t calcByte(uint16_t crc, uint8_t b)
{
    uint32_t i;
    crc = crc ^ (uint32_t)b << 8;
  
    for ( i = 0; i < 8; i++)
    {
      if ((crc & 0x8000) == 0x8000)
        crc = crc << 1 ^ 0x1021;
      else
        crc = crc << 1;
    }
    return crc & 0xffff;
}

uint16_t CRC16(uint8_t *pBuffer, uint32_t length)
{
    uint16_t wCRC16 = 0;
    uint32_t i;
    if (( pBuffer == 0 ) || ( length == 0 ))
    {
        return 0;
    }
    for ( i = 0; i < length; i++)
    {
        wCRC16 = calcByte(wCRC16, pBuffer[i]);
    }
    return wCRC16;
}

uint8_t buf[128];
void loop() {
	uint8_t n,i;
	uint16_t crc;
	n = sprintf(buf, "{\"id\":1,\"name\":\"SN-001\",\"temperature\": 27.45,\"humidity\": 25.36,\"voltage\": 3.88,\"status\": 0}");
	crc = CRC16(buf, n);

	Serial.write(0x7E);
	Serial.write(n);
	for( i = 0; i < n; i++)
	{
		Serial.write(buf[i]);
	}
	Serial.write((uint8_t)(crc >> 8));
	Serial.write((uint8_t)crc);
	delay(2000);
}
```

### 3. 数据模板引擎功能说明

 模板支持的功能：

1. 可以使用原消息中字符串类型的属性值作为属性名
2. 可以使用原消息中的属性值
3. 可以新增自定义的属性名/属性值
4. 可以使用有特定内容的模板，比如时间戳

对原始数据的要求：

1. 格式为 JSON

1. 不能有同名的属性
2. 打算用做模板属性名的字符串属性值需要符合 JSON 属性名命名规范

 模板示例（以 `gw.toml` 中的为例）：

```
# 原始数据示例
"{\"id\":1,\"name\":\"SN-001\",\"temperature\": 27.45,\"humidity\": 25.36,\"voltage\": 3.88,\"status\": 0}"
# 模板
"{<{name}>: [{\"ts\": <#TS#>,\"values\": {\"temperature\": <{temperature}>, \"humidity\": <{humidity}>,\"voltage\": <{voltage}>,\"status\": <{status}>}}]}"
```

以上设置的转换效果为：

```
# 原始数据
{"id":1,"name":"SN-001","temperature": 27.45,"humidity": 25.36,"voltage": 3.88,"status": 0}
# 输出数据
{"SN-001": [{"ts": 1596965153255,"values": {"temperature": 27.45, "humidity": 25.36,"voltage": 3.88,"status": 0}}]}
```

模板注解:

1. `<{name}>` ：取原消息属性 `name` 对应的属性值。例如，需要使用消息 `"temperature": 27.45` 中 `temperature` 的属性值 `27.45` 作为输出数据中的属性值，需要在模板中填写 `<{temperature}>`
2. `<#NAME#>` ：使用模板引擎可以提供的值。例如<#TS#>表示自 EPOCH 以来的秒数；
3. 符合 JSON 属性名命名规范的字符串类型的属性值可以作为模板中的属性名。需要将模板填成 "<{属性名}>" 的形式. 例如, 需要使用消息 `{"name": "SN-001"}`中 `name` 的属性值 `SN-001` 作为输出数据中的属性名, 需要在模板中填写 `<{name}>`。

### 4. 已支持的平台

- x86_64-unknown-linux-gnu
  
  - 需要将子目录 `termios-rs`、`serial-rs`、`ioctl-rs` 切换到 `master` 分支
  
- mips-unknown-linux-uclibc
  - 需要为该目标平台编译 rust：[Cross Compile Rust For OpenWRT](https://qianchenzhumeng.github.io/posts/cross-compile-rust-for-openwrt/)
  - 需要将子目录 `termios-rs`、`serial-rs`、`ioctl-rs` 切换到 `openwrt_cc` 分支
  - 编译命令: 
  
  ```
  #编译 libsqlite3-sys 需要指定交叉编译工具链
  export CC_mips_unknown_linux_uclibc=mips-openwrt-linux-uclibc-gcc
  cargo build --target=mips-unknown-linux-uclibc --release
  ```

默认启用了 rusqlite 的 bundled 特性，libsqlite3-sys 会使用 cc crate 编译 sqlite3，交叉编译时要在环境变量中指定 cc crate使用的编译器(cc crate 的文档中有说明)，否则会调用系统默认的 cc，导致编译过程中出现文件格式无法识别的情况。

### 5. 已知问题

1. 离线数据缓存功能与原始输入数据格式强相关，随意修改原始输入数据会导致该功能不可用，但不会影响在线时的数据发送。
2. 如果使用了数据模板引擎打时间戳的功能（<#TS#>），发布离线时缓存的数据时，输出数据中的时间戳会是发布数据时的时间戳。
