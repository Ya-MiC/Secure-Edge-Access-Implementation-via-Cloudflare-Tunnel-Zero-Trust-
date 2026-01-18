

# Cloudflare Zero Trust 网络访问 (ZTNA) 实施方案

![项目状态](https://img.shields.io/badge/状态-第一阶段%20完成-success)
![支持平台](https://img.shields.io/badge/平台-OpenWrt%20%7C%20Debian-blue)
![中间件](https://img.shields.io/badge/Cloudflare-隧道-orange)
![开源协议](https://img.shields.io/badge/协议-MIT-green)

## 1. 摘要

本项目旨在记录 **Cloudflare Tunnel (cloudflared)** 的部署过程，以建立对本地基础设施的安全、加密入口，无需暴露公网 IP 地址或进行端口转发。主要目标是绕过 CGNAT（运营商级网络地址转换）的限制，并通过 Cloudflare 边缘网络保障本地管理界面的安全访问。

本项目记录了利用 Cloudflare Tunnel 在本地基础设施上部署安全访问的方案。该方案通过建立出站加密隧道，绕过公网 IP 和端口映射限制（穿透 CGNAT），实现了对内网管理界面的 HTTPS 安全访问。

---

## 2. 网络拓扑

该架构采用由本地代理发起的 **反向隧道** 机制。

* **入口域名**: `https://router.yamic189.dpdns.org` (公网)
* **隧道终端**: GL.iNet GL-BE3600 (本地代理)
* **上游服务**: Nginx 服务于 `127.0.0.1:80`

**数据流向**:
`用户客户端` $\rightarrow$ `Cloudflare 边缘节点 (SSL 终止)` $\rightarrow$ `QUIC 协议` $\rightarrow$ `本地代理 (cloudflared)` $\rightarrow$ `本地服务:80`

---

## 3. 第一阶段：边缘路由器部署

**设备规格**:
* **型号**: GL.iNet GL-BE3600
* **架构**: `aarch64` (ARM64)
* **操作系统**: OpenWrt 23.05-SNAPSHOT

### 3.1 二进制文件安装

由于特定 OpenWrt 版本的软件源限制，我们通过 SSH 手动安装了二进制文件。

```bash
# 1. 将 ARM64 架构的二进制文件下载到临时目录
cd /tmp
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64

# 2. 将文件移动到系统可执行文件路径并重命名
mv cloudflared-linux-arm64 /usr/bin/cloudflared

# 3. 赋予执行权限
chmod +x /usr/bin/cloudflared

# 4. 验证安装版本
cloudflared --version
# 预期输出: cloudflared version 2025.11.1...
```

### 3.2 服务持久化策略

由于 OpenWrt 使用 procd/init.d 和 rc.local 而非 systemd，我们将启动逻辑注入到 `/etc/rc.local` 文件中。

配置 (`/etc/rc.local`):

```bash
# 注入启动脚本
cat <<EOF > /etc/rc.local
# Cloudflare Tunnel 开机自启
# 等待15秒以确保网络堆栈初始化完成
sleep 15
/usr/bin/cloudflared tunnel run --token <你的隧道令牌> > /dev/null 2>&1 &

exit 0
EOF

# 赋予脚本执行权限
chmod +x /etc/rc.local
```

**安全提示**: 出于安全考虑，实际的隧道令牌 (Token) 已从本文档中移除。部署时，请将其替换为你的真实令牌。

## 4. 事件报告与故障排查

**事件**: 错误 1033 (隧道错误)
**描述**: 首次连接时，浏览器返回 Cloudflare 错误 1033。

**根本原因分析 (RCA)**:

1.  **协议不匹配**: 隧道被配置为将流量转发到 `https://192.168.8.1:443`。路由器自签名的 SSL 证书导致了握手失败。
2.  **进程终止**: `cloudflared` 进程被手动终止，并未在后台持续运行。

**解决方案**:

1.  **配置更新**: 修改 Cloudflare 控制面板中的设置：
    *   服务类型: HTTP (从 HTTPS 修改)
    *   URL: `127.0.0.1:80` (从 `192.168.8.1` 修改)
2.  **后台运行**: 实现了 `rc.local` 脚本，并通过 `&` 符号确保进程在后台运行。
3.  **服务验证**: 使用 `curl -I http://127.0.0.1:80` 命令验证了本地服务的可访问性。

## 5. 第二阶段：VPS 集成路线图

**目标**: 将一台 RackNerd VPS (Debian 13) 作为独立端点集成到同一个 Zero Trust 网络中。

**计划实施方案**:

*   **安装方式**: 使用原生包管理器 (`apt`) 和 Systemd 服务。
*   **主机名**: `vps.yamic189.dpdns.org`

**命令预览**:

```bash
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared bookworm main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt-get update && sudo apt-get install cloudflared
```

## 附录：仓库配置说明

为保持仓库的整洁与安全，我们应用了以下配置。

### A. .gitignore 配置

创建一个 `.gitignore` 文件以防止敏感数据泄露。

```gitignore
# 安全：永远不要提交凭证信息
*.token
*.key
*.pem
.env
id_rsa

# 系统日志
*.log
/tmp/
cf.log

# 系统元数据
.DS_Store
Thumbs.db
```

### B. 开源协议

本项目采用 MIT 开源协议。

```plaintext
MIT License

Copyright (c) 2026 [Ya-MiC]

特此免费授予任何获得本软件副本及相关文档文件（“软件”）的人不受限制地处理本软件的权利，包括但不限于使用、复制、修改、合并、发布、分发、再许可和/或销售软件副本的权利，并允许获得软件的人这样做，但须符合以下条件：

上述版权声明和本许可声明应包含在软件的所有副本或实质性部分中。

本软件按“原样”提供，不提供任何形式的明示或暗示的保证，包括但不限于对适销性、特定用途的适用性和非侵权性的保证。在任何情况下，作者或版权持有人均不对因软件或软件的使用或其他交易而产生的任何索赔、损害或其他责任承担责任，无论是在合同、侵权还是其他方面。

(完整协议请参见 LICENSE 文件)
```
