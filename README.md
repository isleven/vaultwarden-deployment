# Bitwarden 搭建

## 说明

本文旨在提供简要的技术文档，以在 Linux 系统下使用 Docker 镜像 vaultwarden 和 traefik 快速部署搭建一个自托管的 Bitwarden 服务端站点。

其中 [vaultwarden](https://github.com/dani-garcia/vaultwarden) 原为 bitwarden_rs，是当前最流行的 Bitwarden 服务端的第三方开源实现。[traefik](https://traefik.io) 提供反向代理服务，用于通过 Let's Encrypt 生成域名 SSL 证书及支持 vaultwarden 的 WebSocket。

## 准备

准备一个用于访问 Bitwarden 站点的域名并解析至要部署的服务器/VPS。开放服务器的 80、443 端口并检查是否已有其他进程使用这两个端口。

安装 docker 及 docker-compose（如已安装则略过）：

```bash
curl -sL https://get.docker.com | sh
usermod -aG docker $USER
curl -sL "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# CentOS/RHEL/Fedora 系统下执行
systemctl start docker && systemctl enable docker
```

> 如果在 CentOS 8 系统中执行第一个命令出现错误：
> `Error: Failed to download metadata for repo 'xxx': Cannot prepare internal mirrorlist: No URLs in mirrorlist`
> 则执行下面命令后再重试：
> `sed -i -e "s|mirrorlist=|#mirrorlist=|g" -e "s|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g" /etc/yum.repos.d/CentOS-*`  

## 部署

创建部署目录：

```bash
mkdir /opt/bitwarden
cd /opt/bitwarden
```

创建环境变量文件（**注意修改 `<...>` 部分的值**）：

```bash
cat << 'EOL' > .env
SITE_DOMAIN=<domain-name>  # 准备的域名，如 bw.example.com
SSL_EMAIL=<email-address>  # 生成 SSL 证书时用的 Email，如 example@gmail.com
LOG_MAX_SIZE=10m

# Vaultwarden Configuration
DOMAIN=https://${SITE_DOMAIN}
SIGNUPS_ALLOWED=${SIGNUPS_ALLOWED:-false}
INVITATIONS_ALLOWED=false
WEBSOCKET_ENABLED=true
TZ=Asia/Shanghai

# 以下变量用于配置 Vaultwarden 的 SMTP 邮件发送服务，如无需注册账户的 Email 验证和基于 Email 的两步登录验证功能则可删除或注释
# 推荐使用 Gmail 的 SMTP 发送服务 (https://github.com/dani-garcia/vaultwarden/wiki/SMTP-configuration#googlegmail)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_SSL=false
SMTP_EXPLICIT_TLS=true
SMTP_USERNAME=<gmail-address>
SMTP_PASSWORD=<gmail-app-password>
SMTP_FROM=<gmail-address>
SMTP_FROM_NAME=<sender-name>
SMTP_TIMEOUT=15
EOL
```

创建 Docker Compose 配置文件：

```bash
cat << 'EOL' > docker-compose.yml
version: "3.8"
services:
  traefik:
    image: traefik:v2.6
    container_name: traefik
    restart: unless-stopped
    volumes:
      - ./traefik:/traefik:rw
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - docknet
    ports:
      - 80:80
      - 443:443
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedByDefault=false"
      - "--entrypoints.http.address=:80"
      - "--entrypoints.http.http.redirections.entrypoint.to=https"
      - "--entrypoints.http.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.https.address=:443"
      - "--entrypoints.https.http.tls=true"
      - "--entrypoints.https.http.tls.certResolver=letsencrypt"
      - "--certificatesResolvers.letsencrypt.acme.email=$SSL_EMAIL"
      - "--certificatesResolvers.letsencrypt.acme.storage=/traefik/acme.json"
      - "--certificatesResolvers.letsencrypt.acme.httpChallenge.entryPoint=http"
    logging:
      driver: "json-file"
      options:
        max-size: "$LOG_MAX_SIZE"

  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.vaultwarden-ui.rule=Host(`$SITE_DOMAIN`)"
      - "traefik.http.routers.vaultwarden-ui.service=vaultwarden-service"
      - "traefik.http.services.vaultwarden-service.loadbalancer.server.port=80"
      - "traefik.http.routers.vaultwarden-websocket.rule=Host(`$SITE_DOMAIN`) && Path(`/notifications/hub`)"
      - "traefik.http.routers.vaultwarden-websocket.service=vaultwarden-websocket"
      - "traefik.http.services.vaultwarden-websocket.loadbalancer.server.port=3012"
    volumes:
      - ./vaultwarden/data:/data:rw
    env_file:
      - .env
    networks:
      - docknet
    logging:
      driver: "json-file"
      options:
        max-size: "$LOG_MAX_SIZE"

networks:
  docknet:
    name: docknet
EOL
```

检查域名解析生效后，启动服务：

```bash
SIGNUPS_ALLOWED=true docker-compose up -d
```

这里我们通过在前面添加环境变量 `SIGNUPS_ALLOWED=true` 来临时开启注册。

约一分钟后在浏览器中输入绑定的域名访问 Bitwarden 站点，点击 **Create account** 按钮创建账户。

账户创建完毕，在终端界面执行下面命令重新启动服务以关闭注册（如需开放给其他用户注册则可略过）：

```bash
docker-compose up -d
```

至此 Bitwarden 部署完成。

---

**备注：**

- 默认已在环境变量文件 .env 中关闭了用户注册 (`SIGNUPS_ALLOWED=false`) 和用户邀请 (`INVITATIONS_ALLOWED=false`)。

- 默认未开启管理后台。如果要开启，可在环境变量文件中添加变量 `ADMIN_TOKEN=<token-string>`，重启服务后通过 /admin 路径访问管理面板。

- 环境变量文件中可以自行添加其他 Vaultwarden 的配置项，也可以在管理后台中设置大部分的配置项：[https://github.com/dani-garcia/vaultwarden/wiki/Configuration-overview](https://github.com/dani-garcia/vaultwarden/wiki/Configuration-overview)。

- 修改环境变量文件 .env 或 docker-compose.yml 文件后需再次执行 `docker-compose up -d` 重启服务以使其生效。
