### 1. 功能描述

目前仅支持在 linux 上运行。

**已支持以下功能**：

- 数据上传：从文件读取或者从串口接收 JSON 格式的数据，通过 MQTT 发送给服务器
- 远程控制：可以借助 MQTT 的订阅机制从服务器接收数据，通过串口发送出去
- 网关离线时可以将数据暂时缓存到数据库内，网络连接恢复后再从数据库里面取出来上传
- 支持数据模板功能，即可以根据数据模板重新格式化来自终端的数据、添加自定义属性、预定义属性(例如添加时间戳)等，从而生成新的JSON数据
- 支持日志功能
- 支持 MQTTS(TLS v1.2，v1.3)

**计划开发的功能**：

- 提供配置信息的WEB界面

### 2. 使用示例

在本地启动 MQTT broker，例如使用 mosquitto：

```bash
# 默认监听 1883 端口
mosquitto -v
```

或者修改配置文件 `gw.toml`，指定可用的 MQTT broker：

```toml
[server]
address = "127.0.0.1:1883"
```

#### (1) 从文件读取数据（默认）

修改配置文件（默认是 gw.toml），指定文件，并且将数据接口类型设置为 `text_file`（默认为此配置）：

```toml
[data_if]
#if_name = "/dev/ttyS14"
#if_type = "serial_port"
if_name = "./data_if.txt"
if_type = "text_file"
```

运行网关程序：

```bash
cargo run -- -c gw.toml
```

在文件中写入数据（需要有换行 `'\n'`）：

```bash
echo "{\"id\":1,\"name\":\"SN-001\",\"temperature\": 27.45,\"humidity\": 25.36,\"voltage\": 3.88,\"status\": 0}" > data_if.txt
```

**说明**：网关程序每隔 1 秒读清一次文件，读清后需要手动写入数据。

顺利的话，可以在 mosquitto 的窗口内看到网关发送过去的消息。

#### (2) 从串口读取数据

修改配置文件（默认是 gw.toml），指定串口，并且将数据接口类型设置为 `serial_port`：

```toml
[data_if]
if_name = "/dev/ttyS14"
if_type = "serial_port"
#if_name = "./data_if.txt"
#if_type = "text_file"
```

网关程序：

```bash
cargo run -- -c gw.toml
```

使用外部设备按 [MIN](https://github.com/min-protocol/min) 协议向串口发送数据（ arduino/min 目录内有 Arduino UNO 和 Arduino DUE 的示例，烧录 min 中的程序）：

```
{"id":1,"name":"SN-001","temperature": 27.45,"humidity": 25.36,"voltage": 3.88,"status": 0}
```

如果是 Arduino UNO，只有一个串口，只能用于和网关通信，网关的配置文件中配置接该串口即可。

如果是 Arduino DUE，可以用额外的串口打印调试信息。示例中用到了两个串口，一个是程序烧录串口，用来和网关通信，另一个是 Serial1，打印调试信息，需要额外使用串口转接模块接 TX1、RX1。Serial1 可以改成 SerialUSB（板子上的另一个 microUSB 口）。

顺利的话，网关会收到 Arduino 发送的消息，并且会发送给 mosquitto（可以在 mosquitto 的窗口内看到）。

#### (3) 远程控制

网关已支持远程控制功能。该远程控制不是指可以远程控制网关，而是网关会将服务器发过来的控制命令发送给 MCU，MCU 去响应命令，例如点灯等。

arduino 目录下有 Arduino UNO 和 Arduino DUE 的代码示例（min 子目录），可以通过服务器控制 Arduino 点亮或熄灭 LED。

这里还是以使用 mosquitto 作为 broker 为例：

```bash
# 默认监听 1883 端口
mosquitto -v
```

按照 (2) 中的指引操作。

在另一个终端内使用 mosquitto_pub 按照 gw.toml 里面配置的 `sub_topic`（默认为“ctrl/#”） 发送数据：

```bash
# 点亮 LED
mosquitto_pub -d -h "localhost" -p 1883 -t "ctrl/1" -m "turn_on"
# 熄灭 LED
mosquitto_pub -d -h "localhost" -p 1883 -t "ctrl/1" -m "turn_off"
```

从发布数据到 LED 点亮或熄灭大概会有 3s 左右延时。

#### (4) 使用 TLS

在本地启动 MQTT broker，例如使用 mosquitto：

```bash
mosquitto -c mosquitto.conf
```

配置文件内容（按实际情况修改 ca 文件路径）：

```shell
# mosquitto.conf
log_type error
log_type warning
log_type notice
log_type information
log_type debug

allow_anonymous true

# non-SSL listeners
listener 1883

# server authentication - no client authentication
listener 18885

# 指定 ca 文件路径
cafile ca/ca.crt
certfile ca/server.crt
keyfile ca/server.key
require_certificate false
tls_version tlsv1.2
```

修改 Cargo.toml 文件，开启 `ssl` 特性：

```toml
[features]
default = ["build_bindgen", "bundled", "ssl"]
#default = ["build_bindgen", "bundled"]
```

修改配置文件（默认是 gw.toml），使用 ssl 协议，并指定 ca 文件：

```toml
[server]
#address = "127.0.0.1:1883"
address = "ssl://127.0.0.1:18885"

[tls]
cafile = "ca/ca.crt"
# pem 文件生成方式：cat client.crt client.key ca.crt > client.pem
key_store = "ca/client.pem"
```

编译运行网关程序：

```bash
cargo run -- -c gw.toml
```

#### (5) 连接 ThingsBoard

[待整理]

### 3. 数据模板引擎功能说明

数据模板引擎需要在配置文件中配置，减 gw.toml 内 `[msg]` 部分。

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

```bash
# 原始数据示例
"{\"id\":1,\"name\":\"SN-001\",\"temperature\": 27.45,\"humidity\": 25.36,\"voltage\": 3.88,\"status\": 0}"
# 模板
"{<{name}>: [{\"ts\": <#TS#>,\"values\": {\"temperature\": <{temperature}>, \"humidity\": <{humidity}>,\"voltage\": <{voltage}>,\"status\": <{status}>}}]}"
```

以上设置的转换效果为：

```bash
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
  
- mips-unknown-linux-uclibc
  - 需要为该目标平台编译 rust：[Cross Compile Rust For OpenWRT](https://qianchenzhumeng.github.io/posts/cross-compile-rust-for-openwrt/)
  - 需要编译 openssl
  - 编译命令: 
  
  ```bash
  #编译 libsqlite3-sys 需要指定交叉编译工具链
  export STAGING_DIR=/mnt/f/wsl/OpenWRT/OpenWrt-SDK-ar71xx-for-linux-x86_64-gcc-4.8-linaro_uClibc-0.9.33.2/staging_dir/toolchain-mips_34kc_gcc-4.8-linaro_uClibc-0.9.33.2
  export CC_mips_unknown_linux_uclibc=mips-openwrt-linux-uclibc-gcc
  cargo build --target=mips-unknown-linux-uclibc --release
  ```
  
- arm-unknown-linux-gnueabihf（树莓派）
  
- mipsel-unknown-linux-musl
  - 尚未对串口进行适配
  
- riscv64gc-unknown-linux-gnu

  - 尚未对串口进行适配
  - 编译命令: 
  
  ```bash
  #编译 libsqlite3-sys 需要指定交叉编译工具链
  export CC_riscv64gc_unknown_linux_gnu=riscv64-unknown-linux-gnu-gcc
  cargo build --target=riscv64gc-unknown-linux-gnu --release
  ```
  

默认启用了 rusqlite 的 bundled 特性，libsqlite3-sys 会使用 cc crate 编译 sqlite3，交叉编译时要在环境变量中指定 cc crate使用的编译器(cc crate 的文档中有说明)，否则会调用系统默认的 cc，导致编译过程中出现文件格式无法识别的情况。

为其他平台进行交叉编译时，需要为其单独处理 `paho-mqtt-sys`，该库对应的 github 仓库中有相应的说明。

目录 tools 内有部分平台的编译脚本。

### 5. 交叉编译问题解答

#### (1) ssl 相关

##### a. Dragino LG01-P（mips-openwrt-linux-uclibc）

默认启用了 `paho-mqtt-sys/vendored-ssl` 特性，编译过程中会自动编译自带的 `paho-mqtt-sys` 自带的 openssl 版本，编译成功后，会将其静态链接至最终的可执行文件。这部分是关闭该特性时手动编译 openssl 可能会遇到的问题。`paho-mqtt-sys` 自带的 openssl 版本内没有 mips-unknown-linux-uclibc 的编译配置，需要修改库文件，增加编译配置，或者关闭 `paho-mqtt-sys/vendored-ssl` 特性，手动编译、动态链接。如果是动态链接，需要确保最终的执行环境上有要链接的 openssl 库。

> /mnt/f/wsl/OpenWRT/OpenWrt-SDK-ar71xx-for-linux-x86_64-gcc-4.8-linaro_uClibc-0.9.33.2/staging_dir/toolchain-mips_34kc_gcc-4.8-linaro_uClibc-0.9.33.2/bin/../lib/gcc/mips-openwrt-linux-uclibc/4.8.3/../../../../mips-openwrt-linux-uclibc/bin/ld: cannot find -lssl
> /mnt/f/wsl/OpenWRT/OpenWrt-SDK-ar71xx-for-linux-x86_64-gcc-4.8-linaro_uClibc-0.9.33.2/staging_dir/toolchain-mips_34kc_gcc-4.8-linaro_uClibc-0.9.33.2/bin/../lib/gcc/mips-openwrt-linux-uclibc/4.8.3/../../../../mips-openwrt-linux-uclibc/bin/ld: cannot find -lcrypto

交叉编译 openssl：

```bash
# 设置 STAGING_DIR 环境变量（交叉编译工具链路径）
export STAGING_DIR=/mnt/f/wsl/OpenWRT/OpenWrt-SDK-ar71xx-for-linux-x86_64-gcc-4.8-linaro_uClibc-0.9.33.2/staging_dir/toolchain-mips_34kc_gcc-4.8-linaro_uClibc-0.9.33.2
# 下载、解压源码
wget https://www.openssl.org/source/openssl-1.0.2l.tar.gz
tar zxf openssl-1.0.2l.tar.gz 
cd openssl-1.0.2l
```

```bash
# --prefix 为安装目录
./Configure linux-mips32 no-asm shared --cross-compile-prefix=mips-openwrt-linux-uclibc- --prefix=~/wsl/source/openssl
make
make install
```

将头文件以及共享库复制到交叉编译工具链的相关目录下：

```bash
cp ~/wsl/source/openssl/include/openssl $STAGING_DIR/include -R
cp ~/wsl/source/openssl/lib/*.so* $STAGING_DIR/lib
```

##### b. MT7688（mipsel-unknown-linux-musl）

如果是 OpenWrt，可以通过 make menuconfig 启用 libopenssl（Libraries -> SSL -> libopenssl），然后将头文件以及共享库复制到交叉编译工具链的相关目录下：

```bash
cd ~/openwrt-19.07.2/build_dir/target-mipsel_24kc_musl/openssl-1.1.1d
cp *.so $STAGING_DIR/lib
cp include/openssl/ $STAGING_DIR/include -R
```

> Could not find directory of OpenSSL installation, and this `-sys` crate cannot
>   proceed without this knowledge. If OpenSSL is installed and this crate had
>   trouble finding it,  you can set the `OPENSSL_DIR` environment variable for the
>   compilation process.
>
>   Make sure you also have the development packages of openssl installed.
>   For example, `libssl-dev` on Ubuntu or `openssl-devel` on Fedora.

该信息前面会有一段环境变量相关的内容，根据提示设置相关的变量，例如：

```bash
# 设置环境变量
export MIPSEL_UNKNOWN_LINUX_MUSL_OPENSSL_DIR=$STAGING_DIR
export MIPSEL_UNKNOWN_LINUX_MUSL_OPENSSL_LIB_DIR=$STAGING_DIR/lib
export MIPSEL_UNKNOWN_LINUX_MUSL_OPENSSL_INCLUDE_DIR=$STAGING_DIR/include
# 编译
cargo build --target=mipsel-unknown-linux-musl --release -vv
```

##### c. 芒果派 MQ-R F133（riscv64gc-unknown-linux-gnu）

太老的 openssl 不支持 riscv64 架构，编译会有问题。所以直接使用 SDK 进行编译。在 `Tina-Linux` 的 `menuconfig` 中使能 `openssl` 编译，并启用兼容过时接口选项，设置兼容至 `1.0.0` 版本（依据是 `openssl-1.1.0i/Configure` 中的 `$apitable`，版本号必须是 `$apitable` 中有的，否则编译不过）。之后在编译网关程序时，导出相关的环境变量即可，具体可以参考编译脚本 `tools/build_f133.sh`：

```
#!/bin/bash

# 注意，将 HOME 替换为实际的路径
HOME="/home/dell"
TARGET="riscv64gc-unknown-linux-gnu"
MODE="release"
BIN=target/$TARGET/$MODE/gw
CC="riscv64-unknown-linux-gnu-gcc"
AR="riscv64-unknown-linux-gnu-ar"
CFLAGS="-I$HOME/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/include -I$HOME/Tina-Linux/out/f133-mq_r/compile_dir/target/openssl-1.1.0i/include"
TOOLCHAIN=~/Tina-Linux/out/f133-mq_r/staging_dir/toolchain
STRIP=riscv64-unknown-linux-gnu-strip

export PKG_CONFIG_LIBDIR=~/Tina-Linux/out/f133-mq_r/staging_dir/target/usr/lib/pkgconfig
export PKG_CONFIG_ALLOW_CROSS=1
export CC_riscv64gc_unknown_linux_gnu=$CC
export AR_riscv64gc_unknown_linux_gnu=$AR
export CFLAGS_riscv64gc_unknown_linux_gnu=$CFLAGS
export PATH=$PATH:$TOOLCHAIN/bin
export OPENSSL_INCLUDE_DIR=~/Tina-Linux/out/f133-mq_r/compile_dir/target/openssl-1.1.0i/include
export RISCV64GC_UNKNOWN_LINUX_GNU_OPENSSL_LIB_DIR=~/Tina-Linux/out/f133-mq_r/compile_dir/target/openssl-1.1.0i
cd ..
cargo build --$MODE --target=$TARGET -vv
$STRIP $BIN
echo ""
ls -lh $BIN
```

#### (2) 找不到 libanl

> /mnt/f/wsl/OpenWRT/OpenWrt-SDK-ar71xx-for-linux-x86_64-gcc-4.8-linaro_uClibc-0.9.33.2/staging_dir/toolchain-mips_34kc_gcc-4.8-linaro_uClibc-0.9.33.2/bin/../lib/gcc/mips-openwrt-linux-uclibc/4.8.3/../../../../mips-openwrt-linux-uclibc/bin/ld: cannot find -lanl

按照 [src/CMakeLists.txt: fix build on uclibc or musl](https://github.com/eclipse/paho.mqtt.c/commit/517e8659ab566b15cc409490a432e8935b164de8) 修改 `.cargo/registry/src/crates.rustcc.com-a21e0f92747beca3/paho-mqtt-sys-0.3.0/paho.mqtt.c/src/CMakeLists.txt`

修改后仍然可能因找到了主机的 libanl 报错，如果还报错，按如下方式修改：

```cmake
        #SET(LIBS_SYSTEM c dl pthread anl rt)
		SET(LIBS_SYSTEM c dl pthread rt)
```

#### (3) 找不到 libclang

> thread 'main' panicked at 'Unable to find libclang: "couldn\'t find any valid shared libraries matching: [\'libclang.so\', \'libclang-*.so\', \'libclang.so.*\']

```bash
sudo apt-get install clang libclang-dev
```

#### (4) 找不到 bindings
> thread 'main' panicked at 'No generated bindings exist for the version/target: bindings/bindings_paho_mqtt_c_1.3.2-mips-unknown-linux-uclibc.rs', paho-mqtt-sys/build.rs:102:13

```bash
cargo install bindgen
sudo apt install libc6-dev-i386
cd ~/.cargo/registry/src/crates.rustcc.com-a21e0f92747beca3/paho-mqtt-sys-0.3.0
TARGET=mips-unknown-linux-uclibc bindgen wrapper.h -o bindings/bindings_paho_mqtt_c_1.3.2-mips-unknown-linux-uclibc.rs -- -Ipaho.mqtt.c/src --verbose
```

#### (5) 找不到 C 库头文件

>   debug:clang version: clang version 10.0.0-4ubuntu1
>   debug:bindgen include path: -I/mnt/f/wsl/project/iot_gw/target/mipsel-unknown-linux-musl/release/build/paho-mqtt-sys-0e7cd946c58a7093/out/include
>
>   --- stderr
>   fatal: not a git repository (or any of the parent directories): .git
>   /usr/include/stdio.h:33:10: fatal error: 'stddef.h' file not found
>   /usr/include/stdio.h:33:10: fatal error: 'stddef.h' file not found, err: true
>   thread 'main' panicked at 'Unable to generate bindings: ()', /home/dell/.cargo/registry/src/crates.rustcc.com-a21e0f92747beca3/paho-mqtt-sys-0.3.0/build.rs:139:14
>   note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

安装 clang

```bash
sudo apt-get install clang
```

指定交叉编译工具链，例如：

```bash
export CC_mipsel_unknown_linux_musl=mipsel-openwrt-linux-gcc
export CXX_mipsel_unknown_linux_musl=mipsel-openwrt-linux-g++
```

#### (6) 找不到 MQTTAsync.h

> [paho-mqtt-sys 0.3.0] debug:clang version: clang version 10.0.0-4ubuntu1
> [paho-mqtt-sys 0.3.0] debug:bindgen include path: -I/mnt/f/wsl/project/iot_gw/target/mipsel-unknown-linux-musl/release/b
> uild/paho-mqtt-sys-0e7cd946c58a7093/out/include
> [paho-mqtt-sys 0.3.0] wrapper.h:1:10: fatal error: 'MQTTAsync.h' file not found
> [paho-mqtt-sys 0.3.0] wrapper.h:1:10: fatal error: 'MQTTAsync.h' file not found, err: true
> [paho-mqtt-sys 0.3.0] thread 'main' panicked at 'Unable to generate bindings: ()', /home/dell/.cargo/registry/src/crates
> .rustcc.com-a21e0f92747beca3/paho-mqtt-sys-0.3.0/build.rs:139:14
> [paho-mqtt-sys 0.3.0] note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
> error: failed to run custom build command for `paho-mqtt-sys v0.3.0`

为 mipsel-unknown-linux-musl 编译时遇到了该问题，解决办法是修改 paho-mqtt-sys 的版本：

```toml
[dependencies.paho-mqtt-sys]
default-features = false
#version = "0.3"
version = "0.5"
```

#### (7) pkg-config 报错

> [libudev-sys 0.1.4] thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: "pkg-config has not been configured to support cross-compilation.\n\n                Install a sysroot for the target platform and configure it via\n                PKG_CONFIG_SYSROOT_DIR and PKG_CONFIG_PATH, or install a\n                cross-compiling wrapper for pkg-config and set it via\n                PKG_CONFIG environment variable."', /home/dell/.cargo/registry/src/mirrors.ustc.edu.cn-12df342d903acd47/libudev-sys-0.1.4/build.rs:38:41
> [libudev-sys 0.1.4] note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
> error: failed to run custom build command for `libudev-sys v0.1.4`

安装 `libudev-dev`：

```
sudo apt install libudev-dev
```

设置如下环境变量：

```
export PKG_CONFIG_LIBDIR=/home/dell/wsl/arm-linux-gnueabihf
export PKG_CONFIG_ALLOW_CROSS=1
```

#### (8) 找不到 zlib.h

根据运行环境上的 `libz.so` 的版本，下载对应的 `zlib` 源码，然后在编译时指定头文件路径，例如：

```
CFLAGS="-I/home/dell/wsl/source/libz-1.2.1100+2/libz
```

#### (9) 芒果派 MQR-F133（RISC-V64）生成 `paho-mqtt-sys` 绑定的问题

>  - error: unknown target triple 'riscv64gc-unknown-linux-gnu', please use -triple or -arch
>      thread 'main' panicked at 'libclang error; possible causes include:
>      - Invalid flag syntax
>      - Unrecognized flags
>      - Invalid flag arguments
>      - File I/O errors
>        If you encounter an error missing from this list, please file an issue or a PR!', /home/dell/.cargo/registry/src/mirrors.tuna.tsinghua.edu.cn-df7c3c540f42cdbd/bindgen-0.52.0/src/ir/context.rs:574:15

修改 `bindgen-0.52.0` 的源码，在打印上述信息前的位置将传给 `clang` 的标志打印出来：

```
            println!("{:?}", clang_args);
            clang::TranslationUnit::parse(
                &index,
                "",
                &clang_args,
                &options.input_unsaved_files,
                parse_options,
            ).expect("libclang error; possible causes include:
- Invalid flag syntax
- Unrecognized flags
- Invalid flag arguments
- File I/O errors
If you encounter an error missing from this list, please file an issue or a PR!")
```

得到如下信息：

>   debug:Using bindgen for Paho C
>   debug:clang version: clang version 10.0.0-4ubuntu1
>   debug:bindgen include path: -I/mnt/f/wsl/project/iot_gw/target/riscv64gc-unknown-linux-gnu/release/build/paho-mqtt-sys-b3925b784bd5c394/out/include
>   ["--target=riscv64gc-unknown-linux-gnu", "-I/mnt/f/wsl/project/iot_gw/target/riscv64gc-unknown-linux-gnu/release/build/paho-mqtt-sys-b3925b784bd5c394/out/include", "-isystem", "/usr/local/include", "-isystem", "/usr/lib/llvm-10/lib/clang/10.0.0/include", "-isystem", "/usr/include/x86_64-linux-gnu", "-isystem", "/usr/include", "wrapper.h"]

可以看到，三元组 `riscv64gc-unknown-linux-gnu` 被传给了 `clang`，同时，从 `paho-mqtt-sys-0.5.0/build.rs` 打印的信息可以看到 `clang` 的版本比较老，可能还不支持 `riscv64gc-unknown-linux-gnu` 三元组。最新的 16.0.0 版本是支持的([](https://llvm.org/doxygen/Triple_8h_source.html))。

试一下 `clang-13`:

```
sudo apt-get install clang-13
sudo apt-get install libclang-13-dev
ls -l /usr/bin/clang*
lrwxrwxrwx 1 root root 24 Mar 21  2020 /usr/bin/clang -> ../lib/llvm-10/bin/clang
lrwxrwxrwx 1 root root 26 Mar 21  2020 /usr/bin/clang++ -> ../lib/llvm-10/bin/clang++
lrwxrwxrwx 1 root root 26 Apr 20  2020 /usr/bin/clang++-10 -> ../lib/llvm-10/bin/clang++
lrwxrwxrwx 1 root root 26 Jul  6 20:01 /usr/bin/clang++-13 -> ../lib/llvm-13/bin/clang++
lrwxrwxrwx 1 root root 24 Apr 20  2020 /usr/bin/clang-10 -> ../lib/llvm-10/bin/clang
lrwxrwxrwx 1 root root 24 Jul  6 20:01 /usr/bin/clang-13 -> ../lib/llvm-13/bin/clang
lrwxrwxrwx 1 root root 28 Apr 20  2020 /usr/bin/clang-cpp-10 -> ../lib/llvm-10/bin/clang-cpp
lrwxrwxrwx 1 root root 28 Jul  6 20:01 /usr/bin/clang-cpp-13 -> ../lib/llvm-13/bin/clang-cpp
```

将 `clang` 软链接指向 `clang-13`：

```
sudo rm /usr/bin/clang
sudo ln -s /lib/llvm-13/bin/clang /usr/bin/clang
```

仍然有问题：

> debug:clang version: Ubuntu clang version 13.0.1-2ubuntu2~20.04.1
>   debug:bindgen include path: -I/mnt/f/wsl/project/iot_gw/target/riscv64gc-unknown-linux-gnu/release/build/paho-mqtt-sys-b3925b784bd5c394/out/include
>   ["--target=riscv64gc-unknown-linux-gnu", "-I/mnt/f/wsl/project/iot_gw/target/riscv64gc-unknown-linux-gnu/release/build/paho-mqtt-sys-b3925b784bd5c394/out/include", "-isystem", "/usr/lib/llvm-13/lib/clang/13.0.1/include", "-isystem", "/usr/local/include", "-isystem", "/usr/include/x86_64-linux-gnu", "-isystem", "/usr/include", "wrapper.h"]

从 llvm 源码可以看到，支持 riscv64（https://github.com/llvm/llvm-project/blob/release/13.x/llvm/include/llvm/ADT/Triple.h）：

```
    riscv32,        // RISC-V (32-bit): riscv32
    riscv64,        // RISC-V (64-bit): riscv64
```

但是好像没有 `riscv64gc-unknown-linux-gnu` 的组合。

继续搜索，有人提到，RISC-V 在 `clang` 和 `rustc` 上的三元组不同：https://github.com/rust-lang/rust-bindgen/issues/2136，并且该问题已经得到了解决，但是查看代码时，发现相关提交是 `rust-bindgen` 0.52.0 之后合入的。

修改 `paho-mqtt-sys-0.5.0/Cargo.toml`，将 `bindgen` 的版本修改为 0.60，但还是不行，不知道什么原因，从报错信息中看，调用的仍然是 0.52.0 版。

阅读 `paho-mqtt-sys-0.5.0/build.rs` 源码可知，可以禁用 `build_bindgen` 属性，不要在编译过程中调用 `bindgen` 生成绑定。而是使用 `bindgen` 0.61.0 版手动生成绑定：

```bash
# 删除老版本的 bindgen
cargo uninstall bindgen

# 0.61.0 版本，bindgen 的 crate 改名了
cargo install bindgen-cli

cd ~/.cargo/registry/src/mirrors.tuna.tsinghua.edu.cn-df7c3c540f42cdbd/paho-mqtt-sys-0.5.0

# 但是运行的时候仍然是 bindgen
RUST_BACKTRACE=full TARGET=riscv64gc-unknown-linux-gnu bindgen wrapper.h -o bindings/bindings_paho_mqtt_c_1.3.8-riscv64gc-unknown-linux-gnu.rs -- -Ipaho.mqtt.c/src --verbose
```

有如下报错：

> End of search list.
> thread 'main' panicked at 'assertion failed: `(left == right)`
>   left: `4`,
>  right: `8`: Target platform requires `--no-size_t-is-usize`. The size of `ssize_t` (4) does not match the target pointer size (8)', /home/dell/.cargo/registry/src/mirrors.tuna.tsinghua.edu.cn-df7c3c540f42cdbd/bindgen-0.61.0/codegen/mod.rs:851:25

带上 `--no-size_t-is-usize` 选项：

```
RUST_BACKTRACE=full TARGET=riscv64gc-unknown-linux-gnu bindgen --no-size_t-is-usize wrapper.h -o bindings/bindings_paho_mqtt_c_1.3.8-riscv64gc-unknown-linux-gnu.rs -- -Ipaho.mqtt.c/src --verbose
```

可以生成绑定 `paho-mqtt-sys-0.5.0/bindings/bindings_paho_mqtt_c_1.3.8-riscv64gc-unknown-linux-gnu.rs`：

然后在 `iot_gw/Cargo.toml` 中，禁用 `paho-mqtt-sys/build_bindgen` 特性：

```
[features]
#default = ["build_bindgen", "bundled", "ssl"]
#default = ["build_bindgen", "bundled"]
default = ["bundled"]
build_bindgen = ["paho-mqtt-sys/build_bindgen"]
bundled = ["paho-mqtt-sys/bundled"]
ssl = ["paho-mqtt-sys/vendored-ssl"]
```

进入 `tools` 目录，运行 `build_f133.sh` 即可。

### 6. 网关运行问题解答

#### (1) 数据接口类型未知

> thread 'main' panicked at 'Init data interface failed: DataIfUnknownType'

Cargo.toml 中有关数据接口的特性和网关配置文件内的不一致。

#### (2) 连接错误

> Error connecting to the broker: NULL Parameter: NULL Parameter

使用 ssl 连接 broker，但是没有在 `Cargo.toml` 中启用 `ssl` 特性。

