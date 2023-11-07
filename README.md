# Vaultwarden 搭建

## 说明

本文旨在提供简要的技术文档，以在 Linux 系统下使用 Docker 镜像 vaultwarden 和 traefik 快速部署搭建一个自托管的 Bitwarden 服务端站点 Vaultwarden。

其中 [vaultwarden](https://github.com/dani-garcia/vaultwarden) 是当前最流行的 Bitwarden 服务端的第三方开源实现。[traefik](https://traefik.io) 提供反向代理服务，用于通过 Let's Encrypt 生成域名 SSL 证书及支持 vaultwarden 的 WebSocket。

## 准备

准备一个用于访问 Vaultwarden 站点的域名并解析至要部署的服务器/VPS。开放服务器的 80、443 端口并检查确认没有其他进程使用这两个端口。

安装 docker 及 docker-compose（如已安装则略过）：

```sh
curl -sL https://get.docker.com | sh
usermod -aG docker $USER
curl -sL "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# CentOS/RHEL/Fedora 系统下执行
systemctl start docker && systemctl enable docker
```

> 如果在 CentOS 8 系统中执行第一个命令出现错误：
> 
> `Error: Failed to download metadata for repo 'xxx': Cannot prepare internal mirrorlist: No URLs in mirrorlist`
> 
> 则执行下面命令后再重试：
> 
> `sed -i -e "s|mirrorlist=|#mirrorlist=|g" -e "s|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g" /etc/yum.repos.d/CentOS-*`

## 部署

本文以部署在 /opt/vaultwarden 目录为例，获取本项目文件至该目录：

```sh
git clone https://github.com/isleven/vaultwarden-deployment.git /opt/vaultwarden
```

创建环境变量文件 .env 并**修改其中 `<...>` 部分的变量值**）:

```sh
cp env.template .env
vi .env
```

检查域名解析生效后，启动容器服务：

```sh
SIGNUPS_ALLOWED=true docker-compose up -d
```

> 这里我们通过在启动命令前面添加环境变量 `SIGNUPS_ALLOWED=true` 来临时开启 Vaultwarden 的账户注册。

约两三分钟后在浏览器中输入绑定的域名访问已部署的站点。

点击页面中的 **Create account** 按钮创建账户。

账户创建完毕，在终端界面执行下面命令重新启动服务以关闭注册（如需开放给其他用户注册则可略过）：

```sh
docker-compose up -d
```

至此 Vaultwarden 站点部署完成。

---

## 备注

- 默认在环境变量文件 .env 中关闭了用户注册 (变量 `SIGNUPS_ALLOWED=false`) 和用户邀请 (变量 `INVITATIONS_ALLOWED=false`)。

- 默认未开启管理后台。如果要开启，可在环境变量文件中添加变量 `ADMIN_TOKEN=<token-string>`，重启服务后通过 /admin 路径访问管理面板。

- 环境变量文件中可以自行添加其他 Vaultwarden 的配置项，也可以在管理后台中设置大部分的配置项：[https://github.com/dani-garcia/vaultwarden/wiki/Configuration-overview](https://github.com/dani-garcia/vaultwarden/wiki/Configuration-overview)。

- 修改环境变量文件 .env 或 docker-compose.yml 文件后需再次执行 `docker-compose up -d` 重启服务以使其生效。

- 如部署后需更换站点域名，除将新域名解析至服务器外，还需修改环境变量文件 .env 中变量 `SITE_DOMAIN` 的值，并删除 traefik 目录，最后再重启容器服务。
