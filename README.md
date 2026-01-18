# Secure-Edge-Access-Implementation-via-Cloudflare-Tunnel-Zero-Trust-
Date: January 19, 2026 Author: [Your Name/Username] Status: Phase 1 Completed (Router), Phase 2 Pending (Debian VPS)
1. Abstract / 摘要This document outlines the deployment of Cloudflare Tunnel (formerly Argo Tunnel) to establish secure, encrypted ingress to a local edge device (GL.iNet BE3600 Router) running OpenWrt. The solution bypasses the need for Public IPs and port forwarding (CGNAT traversal) by establishing an outbound connection to Cloudflare's edge network.本文档记录了利用 Cloudflare Tunnel 在边缘设备（OpenWrt 路由器）上部署安全访问的方案。该方案通过建立出站加密隧道，绕过公网 IP 和端口映射限制（穿透 CGNAT），实现了对本地服务的 HTTPS 访问。2. Architectural Principle / 架构原理2.1 Traffic Flow / 流量走向The architecture utilizes a Reverse Tunneling mechanism. Unlike traditional VPNs, the local agent (cloudflared) initiates traffic outbound to Cloudflare, eliminating the need to open inbound firewall ports.Data Path:User Client $\rightarrow$ Internet (HTTPS) $\rightarrow$ Cloudflare Edge (SSL Termination) $\rightarrow$ QUIC Tunnel $\rightarrow$ Router (cloudflared) $\rightarrow$ Local Nginx (127.0.0.1:80)2.2 Security Model / 安全模型Zero Trust: The internal IP (192.168.8.1) is never exposed to the public internet.SSL Offloading: Cloudflare handles the SSL/TLS certificate management automatically.Authentication: The tunnel is authenticated via a unique Token generated in the Cloudflare Zero Trust Dashboard.3. Implementation Details: Phase 1 (Router) / 实施细节Device: GL.iNet GL-BE3600OS: OpenWrt 23.05-SNAPSHOT (Linux)Architecture: aarch64 (ARM64)3.1 Binary Installation / 二进制文件部署Due to the lack of a native package manager in the specific OpenWrt build, a manual binary installation was performed.由于该 OpenWrt 版本软件源限制，采用了手动下载二进制文件的方式安装。Bash# 1. Navigate to temporary directory / 进入临时目录
cd /tmp

# 2. Retrieve the ARM64 binary / 下载对应架构文件
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64

# 3. Relocate to system path and rename / 移动并重命名
mv cloudflared-linux-arm64 /usr/bin/cloudflared

# 4. Assign execution privileges / 赋予执行权限
chmod +x /usr/bin/cloudflared

# 5. Verification / 验证安装
cloudflared --version
# Output: cloudflared version 2025.11.1 ...
3.2 Service Persistence (init.d/rc.local) / 持久化运行Since OpenWrt utilizes procd (and rc.local for simple scripts) rather than systemd, we injected the startup command into /etc/rc.local.由于 OpenWrt 不使用 systemd，我们通过修改启动脚本实现开机自启。Script Injection:Bashcat <<EOF > /etc/rc.local
# Cloudflare Tunnel Auto Start
# Delay to ensure network stack is up / 延迟启动确保网络就绪
sleep 15
/usr/bin/cloudflared tunnel run --token <YOUR_TOKEN_HERE> > /dev/null 2>&1 &

exit 0
EOF

# Grant permission / 赋予脚本执行权限
chmod +x /etc/rc.local
3.3 Configuration Parameters / 配置参数In the Cloudflare Zero Trust Dashboard:Public Hostname: router.yamic189.dpdns.orgService Type: HTTP (Crucial for avoiding SSL handshake loops)Target URL: 127.0.0.1:80Note: Using 127.0.0.1 is preferred over 192.168.8.1 to eliminate routing complexity within the device.注： 指向 127.0.0.1 比内网 IP 更稳定，因为这是回环地址，直接指向本机服务。4. Troubleshooting Log / 故障排查记录Issue: Error 1033 (Tunnel Error)Symptom: User received Cloudflare Error 1033 when accessing the domain.Cause Analysis:Configuration Mismatch: The tunnel was originally pointing to https://192.168.8.1:443. The router's self-signed certificate was rejected by Cloudflare, or the port was unreachable.Process Termination: The cloudflared process was terminated (SIGINT) and not running in the background.Resolution:Changed Service Type to HTTP and Target to 80.Implemented the rc.local script to ensure the daemon runs in the background (&).Verified local service availability via curl -I http://127.0.0.1:80.5. Phase 2: Debian Server Roadmap / 下一阶段规划Target: RackNerd VPS (Debian 13)Goal: Integrate the VPS into the same Cloudflare account but isolate it as a separate "Public Hostname" or separate Tunnel.目标： 将 Debian 服务器接入同个账号，但通过不同子域名访问。5.1 Recommended Strategy / 推荐策略Unlike the Router (which uses rc.local), Debian 13 uses systemd, which allows for a more robust "Service Mode" installation.Debian 13 支持 systemd，因此我们将采用官方推荐的“服务模式”安装，比路由器的脚本方式更稳定。5.2 Implementation Plan / 实施计划Create a New Tunnel (Optional but Recommended):Create a tunnel named RackNerd-VPS to separate "Home" and "Cloud" infrastructure.Why? If the router goes offline, the VPS status remains "Healthy".Installation on Debian:Bash# Step 1: Add Cloudflare GPG key / 添加签名
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

# Step 2: Add Repo / 添加源
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared bookworm main' | sudo tee /etc/apt/sources.list.d/cloudflared.list

# Step 3: Install / 安装
sudo apt-get update && sudo apt-get install cloudflared

# Step 4: Register Service / 注册系统服务
sudo cloudflared service install <NEW_VPS_TOKEN>
Domain Binding:Bind vps.yamic189.dpdns.org (or a specific app subdomain) to localhost:PORT.Next Action: Proceed with Phase 2 on the Debian 13 server.下一步： 开始在 Debian 13 服务器上部署。既然你已经整理好文档，下一步我们做 RackNerd 服务器吗？在 Debian 13 上，我们不需要像在路由器上那样手动改脚本，可以使用更高级的 Systemd 服务 模式。如果你准备好了，请登录你的 Debian 服务器，输入以下命令告诉我结果（确认架构）：uname -m && cat /etc/debian_version
