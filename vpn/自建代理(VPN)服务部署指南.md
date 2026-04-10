# 自建代理(VPN)服务部署指南

> 适用于希望在 Linux 服务器上自建 Shadowsocks 服务，并在手机或电脑上使用代理的用户。

## 使用前提

在开始之前，请先准备：

- 一台 **可正常访问公网的境外 Linux 服务器**
- 服务器的 **公网 IP**
- 服务器的 **root 或 sudo 权限**
- 一台需要配置代理的手机或电脑

> 如果服务器没有公网 IP、网络受限，或云平台安全组未放行端口，客户端将无法连接。

---

## 一、服务端安装

以下示例适用于 `Ubuntu / Debian x86_64`。

### 1. 下载 shadowsocks-rust

```bash
cd /tmp

wget https://github.com/shadowsocks/shadowsocks-rust/releases/download/v1.24.0/shadowsocks-v1.24.0.x86_64-unknown-linux-gnu.tar.xz

tar -xJf shadowsocks-v1.24.0.x86_64-unknown-linux-gnu.tar.xz
```

### 2. 安装二进制

```bash
sudo install -m 755 ssserver /usr/local/bin/ssserver
sudo install -m 755 ssservice /usr/local/bin/ssservice
```

### 3. 检查安装结果

```bash
ssserver -h
ssservice -h
```

---

## 二、创建配置文件

### 1. 创建目录

```bash
sudo mkdir -p /etc/shadowsocks-rust
```

### 2. 创建配置文件

```bash
sudo nano /etc/shadowsocks-rust/config.json
```

填入以下内容：

```json
{
  "server": "0.0.0.0",
  "server_port": 8388,
  "password": "请替换为你的强密码",
  "method": "aes-256-gcm",
  "mode": "tcp_and_udp",
  "timeout": 300
}
```

### 3. 参数说明

- `server`：监听地址，固定写 `0.0.0.0`
- `server_port`：监听端口，例如 `8388`
- `password`：客户端连接密码
- `method`：加密方式，推荐 `aes-256-gcm`
- `mode`：建议使用 `tcp_and_udp`

---

## 三、配置开机自启

### 1. 创建 systemd 服务文件

```bash
sudo nano /etc/systemd/system/shadowsocks.service
```

写入：

```ini
[Unit]
Description=Shadowsocks Rust Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/ssservice server --log-without-time -c /etc/shadowsocks-rust/config.json
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### 2. 启动服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now shadowsocks
```

### 3. 查看状态

```bash
sudo systemctl status shadowsocks
```

---

## 四、放行端口

### 1. Linux 防火墙放行

如果系统启用了 `ufw`：

```bash
sudo ufw allow 8388/tcp
sudo ufw allow 8388/udp
sudo ufw reload
```

### 2. 云服务器安全组放行

在云平台控制台放行以下入站规则：

- TCP `8388`
- UDP `8388`

来源可先设置为：

```text
0.0.0.0/0
```

---

## 五、检查服务是否正常

### 1. 查看监听端口

```bash
ss -lntup | grep 8388
```

正常输出示例：

```bash
udp   UNCONN 0 0 0.0.0.0:8388 0.0.0.0:* users:(("ssservice",pid=xxxx,fd=10))
tcp   LISTEN 0 1024 0.0.0.0:8388 0.0.0.0:* users:(("ssservice",pid=xxxx,fd=9))
```

### 2. 本机测试端口

```bash
nc -vz 127.0.0.1 8388
```

如果返回成功，说明服务已启动。

### 3. 查看服务器公网 IP

```bash
curl ipinfo.io/ip
```

这个 IP 就是客户端要填写的服务器地址。

---

## 六、客户端配置

客户端需要填写以下四项：

- 服务器地址：你的服务器公网 IP
- 端口：`8388`
- 密码：与服务端一致
- 加密方式：`aes-256-gcm`

### Mac

可使用：

- ShadowsocksX-NG
- Clash 系客户端

测试时建议先使用：

- `Global Mode`

### iPhone

常用客户端：

- Shadowrocket

类型选择 `Shadowsocks`，然后填写服务器信息即可。

### Android

常用客户端：

- Shadowsocks Android
- v2rayNG
- Clash Meta for Android

---

## 七、验证是否生效

### 方法 1：查看出口 IP

客户端连接后，访问：

- `https://ipinfo.io`
- `https://ifconfig.me`

如果显示的是服务器公网 IP，说明代理已生效。

### 方法 2：服务器查看日志

```bash
sudo journalctl -u shadowsocks -f
```

客户端访问网页时，如果日志持续有连接记录，说明请求已经到达服务器。

### 方法 3：从外部测试端口

在本地电脑执行：

```bash
nc -vz 服务器公网IP 8388
```

如果返回成功，说明公网到服务器端口是通的。

---

## 八、常见问题

### 1. 报错 `Cannot assign requested address`

通常是配置文件中的 `server` 写错了。

错误示例：

```json
"server": "你的公网IP"
```

正确写法：

```json
"server": "0.0.0.0"
```

### 2. 本机能连，外网不能连

通常优先检查：

- 云服务器安全组是否已放行 `8388`
- Linux 防火墙是否已放行 `8388`
- 客户端填写的是否是正确公网 IP

### 3. 客户端显示已连接，但网页打不开

通常是以下原因：

- 端口不一致
- 密码不一致
- 加密方式不一致
- 安全组未放行

### 4. Mac 上 `curl https://ipinfo.io` 不是服务器 IP

这不一定是故障。常见原因是浏览器已经走代理，但终端中的 `curl` 没有自动走系统代理。可优先用浏览器访问 IP 查询网站确认。

---

## 九、卸载

### 1. 停止服务

```bash
sudo systemctl stop shadowsocks
sudo systemctl disable shadowsocks
```

### 2. 删除服务文件

```bash
sudo rm -f /etc/systemd/system/shadowsocks.service
sudo systemctl daemon-reload
```

### 3. 删除配置和程序

```bash
sudo rm -rf /etc/shadowsocks-rust
sudo rm -f /usr/local/bin/ssserver
sudo rm -f /usr/local/bin/ssservice
```

---

## 十、最小配置示例

```json
{
  "server": "0.0.0.0",
  "server_port": 8388,
  "password": "请替换为你的强密码",
  "method": "aes-256-gcm",
  "mode": "tcp_and_udp",
  "timeout": 300
}
```
