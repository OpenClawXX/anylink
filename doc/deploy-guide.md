# AnyLink 部署流程与操作指南

本文面向 Linux 服务器部署 AnyLink（管理后台 + VPN 接入服务）。内容包含：部署前检查、Docker 部署、二进制 + systemd 部署、K8s 部署要点、常见操作与排障。

## 0. 部署前检查

### 0.1 系统与权限

- Linux 主机（推荐 Ubuntu 20/22/24、Debian 11/12、CentOS/Rocky 系）。
- 需要 root 权限（AnyLink 会创建 TUN/TAP 设备、配置转发、写入 iptables 规则）。
- 内核需支持 TUN/TAP：

```bash
ls -l /dev/net/tun
modprobe tun
```

### 0.2 网络与端口

- TCP 443：VPN 接入（TLS）
- UDP 443：可选 DTLS（配置开启后使用）
- TCP 8800：管理后台

确保安全组/防火墙放通以上端口。

### 0.3 证书与域名（建议）

- 正式环境建议使用受信任证书（Let’s Encrypt / 商业证书）。
- 配置文件中对应字段：
  - `cert_file`：证书 PEM（包含证书链）
  - `cert_key`：证书私钥 KEY

## 1. Docker Compose 部署（推荐）

### 1.1 准备配置目录（可选但推荐）

建议把配置持久化到宿主机目录，例如 `/opt/anylink/conf`：

```bash
mkdir -p /opt/anylink/conf
```

把项目中的配置模板拷贝过去（首次部署可直接用示例再改）：

```bash
cp -r conf/* /opt/anylink/conf/
```

### 1.2 使用 docker-compose 启动

项目自带 compose 模板参考：[docker-compose.yaml](file:///workspace/deploy/docker-compose.yaml)

示例（推荐把 conf 挂载出来，便于修改与持久化）：

```yaml
services:
  anylink:
    image: bjdgyc/anylink:latest
    container_name: anylink
    restart: always
    privileged: true
    ports:
      - "443:443"
      - "8800:8800"
      - "443:443/udp"
    command:
      - --conf=/app/conf/server.toml
    volumes:
      - /opt/anylink/conf:/app/conf
```

启动：

```bash
docker compose up -d
```

### 1.3 首次启动的“初始化后退出”行为说明

镜像入口脚本会在检测到 `/app/conf/profile.xml` 不存在时，自动把容器内置的默认配置复制出来后退出（需要你重启容器）。入口脚本逻辑见：[docker_entrypoint.sh](file:///workspace/docker/docker_entrypoint.sh)

操作建议：

1. 第一次 `up -d` 后查看日志，若提示“配置文件初始化完成后，容器会强制退出，请重新启动容器”，属于正常初始化流程。
2. 确认宿主机 `/opt/anylink/conf` 已生成配置文件并按需修改（证书、网段、后台密码等）。
3. 再次启动：

```bash
docker compose up -d
```

### 1.4 访问与默认账号

- 管理后台：https://你的域名或IP:8800
- 默认账号密码以配置文件为准（`admin_user` / `admin_pass` / `admin_otp`）

## 2. 二进制 + systemd 部署（无 Docker）

### 2.1 目录结构建议

```text
/usr/local/anylink-deploy/
  anylink
  conf/
  deploy/
  index_template/
  log/
```

### 2.2 配置文件

主配置文件：`conf/server.toml`（示例见项目：`server/conf/server.toml`）

常用项（务必核对）：

- `cert_file` / `cert_key`：证书路径
- `server_addr`：VPN TCP 监听（默认 `:443`）
- `admin_addr`：后台监听（默认 `:8800`）
- `ipv4_master`：出口网卡名（常见为 `eth0` 或 `ensXXX`）
- `ipv4_cidr` / `ipv4_gateway` / `ipv4_start` / `ipv4_end`：虚拟地址池
- `iptables_nat`：是否自动添加 NAT（一般 true）

### 2.3 systemd 服务

项目自带 service 文件参考：[anylink.service](file:///workspace/deploy/anylink.service)

安装步骤：

```bash
cp deploy/anylink.service /etc/systemd/system/anylink.service
systemctl daemon-reload
systemctl enable anylink
systemctl start anylink
systemctl status anylink -l
```

日志查看：

```bash
journalctl -u anylink -f
```

### 2.4 启动失败常见原因

- 没有 `/dev/net/tun`：执行 `modprobe tun`，并确保宿主机允许创建 tun 设备。
- `net.ipv4.ip_forward` 未开启：执行 `sysctl -w net.ipv4.ip_forward=1`。
- iptables 权限/兼容性问题：部分发行版可能需要 legacy iptables（容器环境可用环境变量 `IPTABLES_LEGACY=on`，二进制模式需自行处理）。

## 3. Kubernetes 部署要点（可选）

项目有示例清单：[deployment.yaml](file:///workspace/deploy/deployment.yaml)

关键点：

- `securityContext.privileged: true`（需要创建 TUN、操作 iptables）
- 端口需要同时暴露 TCP 443、TCP 8800、UDP 443
- 若涉及真实网络转发与路由，通常还需要结合 hostNetwork、CNI、节点路由策略做设计（否则仅“容器能起”不代表“VPN 能通”）

## 4. 常用运维操作

### 4.1 修改后台密码 / JWT secret / OTP

使用二进制自带工具（示例）：

```bash
./anylink tool -p 123456
./anylink tool -s
./anylink tool -o
```

将生成的值写入 `conf/server.toml` 对应字段后重启服务。

### 4.2 重启服务

- Docker：

```bash
docker compose restart
```

- systemd：

```bash
systemctl restart anylink
```

## 5. 验收清单（建议）

- 能打开管理后台并登录
- 新建用户/组/策略能保存（数据库可写）
- 客户端可连接到 `:443` 并分配到地址池 IP
- 开启 `iptables_nat` 后能访问内网资源（路由与防火墙规则符合预期）

