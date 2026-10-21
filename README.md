# SCUT 有线与无线校园网客户端

面向 Linux 的华南理工大学校园网认证客户端，基于 [scutclient/scutclient](https://github.com/scutclient/scutclient) 修改。此分支保留有线 Dr.com / 802.1X 认证，并加入 C++ 命令行实现、无线门户认证和 systemd 服务配置。

## 当前功能与上游区别

| 项目 | 源项目 | 本分支 |
|---|---|---|
| 有线认证 | C 实现，程序名 `scutclient` | C++17 重构，程序名 `DrcomScutClient`；提供认证、重试、注销和日志功能 |
| 命令行 | `getopt` 参数解析 | 使用 [clipp](https://github.com/muellan/clipp)，网卡参数为 `--interface` |
| 无线门户 | 无本分支的 `wireless/` 模块 | 新增独立 `wireless` 程序，包含门户重定向、连接参数提取和登录请求流程，已由作者长期使用 |
| 服务管理 | 提供 OpenWrt 集成文件 | 另加 systemd service / timer；账号密码从仓库外的环境文件读取 |
| 构建与维护 | 上游构建及打包流程 | CMake 构建两个程序，增加卸载目标、凭据配置示例和本机维护说明 |

这是独立维护的个人分支，尚未同步上游全部后续提交。对照时间：2026-10-21；上游来源与当前分支分别见 [源项目](https://github.com/scutclient/scutclient) 和 [JHees/scut-network-client](https://github.com/JHees/scut-network-client)。

## 编译

需要 Linux、C++17 编译器、CMake、libcurl、Boost 头文件和 `clipp.h`。Debian / Ubuntu 可先安装依赖：

```bash
sudo apt install build-essential cmake curl libcurl4-openssl-dev libboost-dev
curl -fL https://raw.githubusercontent.com/muellan/clipp/v1.2.3/include/clipp.h -o /tmp/clipp.h
sudo install -m 644 /tmp/clipp.h /usr/local/include/clipp.h

git clone https://github.com/JHees/scut-network-client.git
cd scut-network-client
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_POLICY_VERSION_MINIMUM=3.5
cmake --build build -j2
```

`clipp` 是单头文件依赖，目前未随仓库提供。`CMAKE_POLICY_VERSION_MINIMUM` 用于兼容现有构建文件与 CMake 4；本次已在 CMake 4.2.3 上验证配置命令。

构建产物为 `build/DrcomScutClient` 和 `build/wireless`。只构建有线程序时可使用 `cmake --build build --target DrcomScutClient -j2`。

## 使用

有线网卡需先接入适用的校园网；认证程序访问原始网络套接字，需要相应权限：

```bash
./build/DrcomScutClient --help
sudo ./build/DrcomScutClient --username '<校园网账号>' --password '<校园网密码>' --interface eth0
```

将 `eth0` 换成实际有线网卡名称。其他参数以有线程序的 `--help` 为准。

无线程序要求系统已经连接 Wi-Fi，仅尝试门户认证：

```bash
./build/wireless --username '<校园网账号>' --password '<校园网密码>' --interface wlan0
```

无线门户认证已由作者长期使用。该实现面向 SCUT 门户接口，在其他网络环境使用时需核对门户地址和参数。

## 可选的 systemd 服务

默认安装前缀为 `/usr/local`。`sudo cmake --install build --config Release` 会安装有线程序和 service / timer 文件，不会自动启用它们，也不会安装无线程序。

使用服务前，将 [`scutclient.env.example`](scutclient.env.example) 复制为 `/etc/scutclient.env`，填入自己的账号密码，并设为 root 所有、权限 `600`；同时核对服务模板中的 `eth0`。环境文件用于避免把真实凭据写进仓库，程序仍通过命令行参数接收凭据。

[`DrcomScutClient.service`](DrcomScutClient.service) 负责有线认证，失败后重试；[`DrcomScutClient.timer`](DrcomScutClient.timer) 配置为启动约 10 秒后及每天 06:05 触发服务。仅在需要认证的设备上按需启用，当前维护用树莓派没有安装或启用这些旧任务。

## 功能边界与本地资料

- **未实现自动连接 Wi-Fi、有线 / 无线切换、路由切换或热点管理。** `wireless/` 中相关流程仍有 TODO，`addRouteRable()` 当前只解析并打印地址。
- OpenWrt 打包文件仍保留上游下载地址与 `scutclient` 可执行文件名；直接打包此分支前需要适配，不能照搬旧版 OpenWrt 步骤。
- [`docs/NETWORK-MAINTENANCE.md`](docs/NETWORK-MAINTENANCE.md) 是特定树莓派的历史维护记录，不是通用部署配置。
- `disabled-tasks/` 保存本机停用任务及说明，`network-backups/` 保存历史配置；它们可能含私有凭据，均被 Git 忽略，不随公开仓库发布。真实账号、密码、认证日志和构建产物也不要提交。

## 许可证

沿用上游的 [GNU AGPL v3](COPYING)，保留原项目作者与版权声明。
