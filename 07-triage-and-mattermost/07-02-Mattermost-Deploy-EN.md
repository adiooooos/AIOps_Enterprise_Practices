# Mattermost Installation and Configuration

> **Status**: 20260609 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Chapter Overview

| # | Dimension | Description |
| --- | --- | --- |
| ① | **Method 1** | Online install: `podman-compose` + `compose.yml` |
| ② | **Method 2** | Offline install: `docker save` / `scp` / `podman load` |
| ③ | **Port** | `8065/tcp` (firewalld) |
| ④ | **Webhook** | `AllowedUntrustedInternalConnections` allow `10.0.0.0/8` |

---

## 1. Installation Method 1: Online Install

### 1.1 Software Preparation

1）软件准备

```bash
sudo dnf install -y python3-pip
sudo pip3 install podman-compose
podman-compose --version
sudo firewall-cmd --add-port=8065/tcp --permanent && firewall-cmd --reload && firewall-cmd --list-ports
```

### 1.2 Mattermost Installation

2）Mattermost安装

```bash
# 安装前准备
mkdir mattermost && cd mattermost
mkdir -p ./volumes/app/config
mkdir -p ./volumes/app/data
mkdir -p ./volumes/app/logs
mkdir -p ./volumes/app/plugins
mkdir -p ./volumes/app/client/plugins
mkdir -p ./volumes/db/data
# podman compose准备
```

```text
[root@mattermost mattermost]# more compose.yml
```

```yaml
version: "3.8"

services:
  # PostgreSQL 数据库服务
  postgres:
    image: docker.io/postgres:15-alpine
    container_name: mattermost-postgres
    restart: unless-stopped
    volumes:
      # :Z 标记会自动处理 SELinux 的文件权限问题
      - ./volumes/db/data:/var/lib/postgresql/data:Z
    environment:
      - POSTGRES_USER=mmuser
      - POSTGRES_PASSWORD=redhat
      - POSTGRES_DB=mattermost
    networks:
      - mattermost-net

  # Mattermost 应用服务
  mattermost:
    image: docker.io/mattermost/mattermost-team-edition:latest
    container_name: mattermost-app
    restart: unless-stopped
    depends_on:
      - postgres
    ports:
      - "8065:8065" # 将主机的 8065 端口映射到容器的 8065 端口
    volumes:
      # :Z 标记是 RHEL/Podman 环境下的关键
      - ./volumes/app/config:/mattermost/config:Z
      - ./volumes/app/data:/mattermost/data:Z
      - ./volumes/app/logs:/mattermost/logs:Z
      - ./volumes/app/plugins:/mattermost/plugins:Z
      - ./volumes/app/client/plugins:/mattermost/client/plugins:Z
    environment:
      # 数据库连接设置 - 必须与上面的 postgres 服务配置匹配
      - MM_SQLSETTINGS_DRIVERNAME=postgres
      - MM_SQLSETTINGS_DATASOURCE=postgres://mmuser:redhat@postgres:5432/mattermost?sslmode=disable&connect_timeout=10
      # 站点URL - 启动后需要在后台修改为您的域名或IP
      - MM_SERVICESETTINGS_SITEURL=http://localhost:8065
    networks:
      - mattermost-net

networks:
  mattermost-net:
    driver: bridge
```

```bash
#安装Mattermost
sudo chown -R 2000:2000 ./volumes

# 解决Mattermost发送信息失败的问题：
Outgoing Webhook POST failed
app/webhook.go:153
2025-09-21 08:40:56.032 Z
error
Outgoing Webhook POST failed
app/webhook.go:153
问题根源：
Outgoing Webhook POST failed 之谜解开了：当 Mattermost 应用本身尝试向外发送请求时，它会检查自己的 config.json 配置文件。默认情况下，为了安全，它会阻止自己向未明确信任的内部IP地址（如 10.x.x.x）发送数据。您将网络地址加入白名单后，这个应用层面的安全限制就被解除了。
podman-compose up -d
podman ps
podman logs -f mattermost-app

# 更新配置
[root@diagnose mattermost]# vi  ./volumes/app/config/config.json
# 为了允许向您的 10.x.x.x 网络发送请求，它的值应该包含 "10.0.0.0/8"。例如：
"AllowedUntrustedInternalConnections": "10.0.0.0/8 127.0.0.1/8",
#  再次重启下容器即可
```

---

## 2. Installation Method 2: Offline Install

Mattermost无法在线安装的解决办法如下：

### 2.1 Download Images on an Internet-Connected Machine

```text
# 在可以连接外网的机器上执行以下命令下载软件
PS C:\Users\zhang> docker pull postgres:15-alpine
15-alpine: Pulling from library/postgres
a8d05d343dd2: Pull complete
…
a64d9570aeee: Pull complete
Digest: sha256:fb9065b6e3e213bdc07edd372a5b2a26245840b7fb65d1fd8b6700106d51805c
Status: Downloaded newer image for postgres:15-alpine
docker.io/library/postgres:15-alpine
PS C:\Users\zhang> docker pull mattermost/mattermost-team-edition:latest
latest: Pulling from mattermost/mattermost-team-edition
526604835308: Pull complete
…
ef49c20a7b35: Pull complete
Digest: sha256:425b1e065a3433f4a7b10d8091fe83b3459c008647cc1e3c3707974832d2ef30
Status: Downloaded newer image for mattermost/mattermost-team-edition:latest
docker.io/mattermost/mattermost-team-edition:latest
PS C:\Users\zhang> docker save -o mattermost_bundle.tar postgres:15-alpine mattermost/mattermost-team-edition:latest
PS C:\Users\zhang> scp mattermost_bundle.tar root@10.210.65.14:/root/mattermost/
The authenticity of host '10.210.65.14 (10.210.65.14)' can't be established.
ED25519 key fingerprint is SHA256:c4ugkrw1T7I8xD3j+BdFQXLlmV6WcuEj4XAHp+Rrbds.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
Warning: Permanently added '10.210.65.14' (ED25519) to the list of known hosts.
root@10.210.65.14's password:
mattermost_bundle.tar                                                                         100%  491MB  11.3MB/s   00:43
```

### 2.2 Import and Start on the Mattermost Host

```text
# 在Mattermost机器上执行以下操作
[root@mattermost mattermost]# cd /root/mattermost/
[root@mattermost mattermost]# chmod 775 mattermost_bundle.tar
[root@mattermost mattermost]# sudo podman load -i mattermost_bundle.tar
Getting image source signatures
Copying config 4ba28b1f75 done   |
…
Copying config f6cccf3055 done   |
Writing manifest to image destination
Loaded image: localhost/postgres:15-alpine
Loaded image: localhost/mattermost/mattermost-team-edition:latest
[root@mattermost mattermost]# podman images
REPOSITORY                                    TAG         IMAGE ID      CREATED       SIZE
localhost/mattermost/mattermost-team-edition  latest      f6cccf30550c  2 days ago    603 MB
localhost/postgres                            15-alpine   4ba28b1f75f3  5 months ago  271 MB
[root@mattermost mattermost]# podman tag localhost/postgres:15-alpine docker.io/library/postgres:15-alpine
[root@mattermost mattermost]# podman tag localhost/mattermost/mattermost-team-edition:latest docker.io/mattermost/mattermost-team-edition:latest
[root@mattermost mattermost]# podman image list
REPOSITORY                                    TAG         IMAGE ID      CREATED       SIZE
docker.io/mattermost/mattermost-team-edition  latest      f6cccf30550c  2 days ago    603 MB
localhost/mattermost/mattermost-team-edition  latest      f6cccf30550c  2 days ago    603 MB
docker.io/library/postgres                    15-alpine   4ba28b1f75f3  5 months ago  271 MB
localhost/postgres                            15-alpine   4ba28b1f75f3  5 months ago  271 MB
[root@mattermost mattermost]#
^C
[root@mattermost mattermost]# podman-compose up -d
6b03f7fb7fcdb71d31f35c61665633af2d4bb00a4e77be536a6df5d38062d7db
3c6d1caf5ae6948a1d585ae6d22de733fae0de961534d8de6e757240fc41b8c2
mattermost-postgres
mattermost-app

# 更新配置
[root@diagnose mattermost]# vi  ./volumes/app/config/config.json
# 为了允许向您的 10.x.x.x 网络发送请求，它的值应该包含 "10.0.0.0/8"。例如：
"AllowedUntrustedInternalConnections": "10.0.0.0/8 127.0.0.1/8",
#  再次重启下容器即可（先关app再关数据库，先启数据库，再启app）
[root@mattermost mattermost]# podman start mattermost-postgres
mattermost-postgres
[root@mattermost mattermost]# podman start mattermost-app
mattermost-app
```
