# 内网穿透-利用 FRP 实现内网穿透，让家里的服务器秒变“永远在线”的云主机


> 利用 FRP 实现内网穿透，让家里的服务器秒变“永远在线”的云主机

# 0. 需求场景
👉在日常工作和生活中，我们常常需要远程访问内网资源，FRP 就是一个高效解决方案。
* 异地访问公司内部系统，如 OA、ERP、GitLab、文件服务器等。
* 在外网环境下访问家庭 NAS、影音库等资源。
* 开发调试与测试场景，如支付回调/Webhook 测试，微信支付、支付宝等第三方平台需要回调本地 API。
* IoT 与智能设备管理，例如远程访问家中的 HomeAssistant、树莓派等。
* 联机游戏
市面上也有成熟的产品，如`花生壳`、`natapp`、`ngrok`、`cpolar`等，但这些都有些限制，不如自己搭建的自由，所以就有了本文的主角`FRP`。

# 1. 搭建前准备
👉 在动手前，我们需要准备一台 **具有公网 IP** 的云主机，作为 FRP 服务端。
* 本文示例系统：Ubuntu 24.04。

# 2. 搭建步骤
### 2.1 FRP 原理简介
👉 先理解 FRP 的工作原理，再动手会更轻松。
FRP **采用 C/S 模式**：
* 服务端（frps）部署在具备公网 IP 的云主机上。
* 客户端（frpc）部署在需要被访问的内网设备上。
通过访问服务端暴露的端口，即可反向代理到内网服务。

### 2.2 服务端(frps)搭建
👉 先在云主机上搭建 FRP 服务端，它是整个穿透的核心。
使用wget命令下载压缩包，注意📢需要根据cpu架构下载其对应的安装包
```sh
wget https://github.com/fatedier/frp/releases/download/v0.64.0/frp_0.64.0_linux_amd64.tar.gz
```sh
解压
```sh
tar -zxvf frp_0.64.0_linux_amd64.tar.gz
cd frp_0.64.0_linux_amd64/
```
配置文件调整, `vim  frps.toml`
```yml
#  frps服务器用于接收客户端连接的端口
bindPort = 7000
# 以下是dashboard配置，不需要dashboard可以不配置。
# 默认为 127.0.0.1，如果需要公网访问，需要修改为 0.0.0.0。
webServer.addr = "0.0.0.0"
webServer.port = 7500
# dashboard 用户名密码
webServer.user = "your_name"
webServer.password = "your_password"
```
将可执行文件和配置文件移动到指定目录
```sh
sudo mv frps /usr/local/bin/
sudo mv frps.toml  /usr/local/etc/
```
创建 `frps.service` 文件,`vim /lib/systemd/system/frps.service`
```sh
[Unit]
Description=frps server daemon
Documentation=https://github.com/fatedier/frp
After=network-online.target

[Service]
ExecStart=/usr/local/bin/frps -c /usr/local/etc/frps.toml
Type=simple
User=nobody
Group=nogroup
WorkingDirectory=/tmp
Restart=on-failure
RestartSec=60s

[Install]
WantedBy=multi-user.target
```
使用 systemd 命令管理 frps 服务
```sh
# 启动frp
sudo systemctl start frps
# 停止frp
sudo systemctl stop frps
# 重启frp
sudo systemctl restart frps
# 查看frp状态
sudo systemctl status frps
```
🔥 设置 frps 开机自启动
```sh
sudo systemctl enable frps
```


### 2.3 客户端(frpc)搭建
👉 再在需要被访问的内网机器（家用电脑、NAS、树莓派等）上部署 FRP 客户端。
> 与服务端(frps)搭建稍微有少量区别；在需要被访问的内网机器上部署`frpc`,如 想让家里的电脑秒变永远在线的云主机(断电,断网除外)，此时就需要把`frpc`家里的电脑上。

使用wget命令下载压缩包，注意📢需要根据cpu架构下载其对应的安装包

```sh
wget https://github.com/fatedier/frp/releases/download/v0.64.0/frp_0.64.0_linux_amd64.tar.gz
```
解压
```sh
tar -zxvf frp_0.64.0_linux_amd64.tar.gz
cd frp_0.64.0_linux_amd64/
```
配置文件调整,配置文件调整, `vim  frpc.toml`
```yml
serverAddr = "服务端公网IP"
serverPort = 7000

# ssh端口
[[proxies]]
name = "ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6001

# mysql端口
[[proxies]]
name = "mysql"
type = "tcp"
localIP = "127.0.0.1"
localPort = 3306
remotePort = 6002
```
将可执行文件和配置文件移动到指定目录
```sh
sudo mv frpc /usr/local/bin/
sudo mv frpc.toml  /usr/local/etc/
```
创建 `frpc.service` 文件,`vim /lib/systemd/system/frpc.service`
```sh
[Unit]
Description=frpc server daemon
Documentation=https://github.com/fatedier/frp
After=network-online.target

[Service]
ExecStart=/usr/local/bin/frpc -c /usr/local/etc/frpc.toml
Type=simple
User=nobody
Group=nogroup
WorkingDirectory=/tmp
# 🔥 失败后20s重试，官网文档没有
Restart=on-failure
RestartSec=20s

[Install]
WantedBy=multi-user.target
```
使用 systemd 命令管理 frpc 服务
```sh
# 启动frp
sudo systemctl start frpc
# 停止frp
sudo systemctl stop frpc
# 重启frp
sudo systemctl restart frpc
# 查看frp状态
sudo systemctl status frpc
# 重新加载服务的配置文件
sudo systemctl daemon-reload
```
🔥 设置 frpc 开机自启动
```sh
sudo systemctl enable frpc
```

### 2.4 云主机安全组放行端口
👉 服务器端口必须在云厂商安全组中放行，否则外网无法访问。
笔者买的是腾讯云的云主机，需要在腾讯云云主机管理页面开启 
* frps服务器接收客户端连接的端口`7000`
* frps服务器dashboard控制台`7500` 
* frpc客户端ssh代理端口`6001`，
* frpc客户端MySQL代理端口`6002`

大家根据自己的需要开放对应端口即可，为了安全，依据最小化暴露端口原则，不要开启无关端口。

### 2.5 验证配置
👉 配置完成后，用 SSH 测试最直观。
找另外一台linux或者mac电脑或者有bashshell的windows电脑，在终端执行

```sh
ssh  -p 端口号 客户端用户名@服务端ip
```
如果能成功登录到家里的电脑，那么恭喜你，内网搭建成功。

# 3. 结语：如何让家里的电脑“永远在线”？
👉 只要考虑到电源、网络、服务自动化，就能让家里的电脑像云主机一样 7×24 小时运行。
### 3.1 异常情况及解决方案
👉 针对断电、断网、服务异常等情况，逐一设定自动恢复方案。
* 断电后自动开机:在 BIOS 中开启通电自启（AC Power Recovery），不同品牌路径不同，大致为：BIOS 设置 -> System BIOS -> System Security -> AC Power Recovery -> ON
* 系统启动后自动运行 frpc: 已通过 systemd 配置开机自启。
* 网络未准备好或 frpc 启动失败: frpc.service 中已设置失败重试，确保网络恢复后可自动重连。详情见`frpc.service`文件
* 极端情况( frp 本身故障): 可安装 Zerotier、RustDesk、向日葵等远程工具，与 FRP 互为主备，提高可用性。

# 4. 参考链接
* [FRP官网](https://gofrp.org/zh-cn/)

