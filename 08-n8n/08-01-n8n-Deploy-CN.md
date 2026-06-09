# n8n 软件包准备

> **状态**：20260609 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 说明 |
| --- | --- | --- |
| ① | **方式 1** | 在线部署：`podman pull` + `podman run` |
| ② | **方式 2** | 离线部署：`docker save` / `scp` / `docker load` |
| ③ | **端口** | `8080`（`N8N_PORT`） |
| ④ | **首次访问** | 创建或指定管理员账号 |

---

## 一、在线部署

### 1.1 下载 registry 镜像

```text
### 下载registry镜像[root@aap26-jw ~]# podman pull registry
[root@n8n ~]# podman pull n8nio/n8n:latest
✔ docker.io/n8nio/n8n:latest
Trying to pull docker.io/n8nio/n8n:latest...
Getting image source signatures
Copying blob 81d49b7a1dbb done   |
Copying blob 22b37da5853a done   |
Copying blob cb0e99e1c627 done   |
Copying blob 59ef0fb7d7f5 done   |
Copying blob ea2debf3e1ac done   |
Copying blob d1520cba0de0 done   |
Copying blob eb7d9231e3ec done   |
Copying blob 4f4fb700ef54 done   |
Copying blob 4f4fb700ef54 done   |
Copying blob c309291cc0d9 done   |
Copying blob f74dae35a608 done   |
Copying blob 3be0550b41f6 done   |
Copying config 05ec79d62b done   |
Writing manifest to image destination
05ec79d62bc4c2ee6b7b53449e8e90944439f7a6967d77e78fcc9d1c409d5e07
```

### 1.2 执行安装

```text
### 执行安装：
```

```bash
[root@n8n ~]# podman run -d --name n8n --restart unless-stopped \
  -p 8080:8080 \
  -v n8n_data:/home/node/.n8n \
  -v /opt/n8n/files:/tmp \
  -e N8N_LISTEN_ADDRESS=0.0.0.0 \
  -e N8N_PORT=8080 \
  -e N8N_PROTOCOL=http \
  -e N8N_HOST=10.210.65.103 \
  -e N8N_SECURE_COOKIE=false \
  n8nio/n8n:latest
```

```text
7f58394e0038face9ef86b82dbddfd37de017a2b6ebad51f142e6922291fc902

[root@n8n ~]# podman ps -a
CONTAINER ID  IMAGE                       COMMAND     CREATED        STATUS        PORTS                             NAMES
7f58394e0038  docker.io/n8nio/n8n:latest              7 seconds ago  Up 7 seconds  0.0.0.0:8080->8080/tcp, 5678/tcp  n8n

[root@n8n ~]# podman logs -f 7f58394e0038
No encryption key found - Auto-generating and saving to: /home/node/.n8n/config
Initializing n8n process
n8n ready on 0.0.0.0, port 8080
```

容器启动后，通过域名或IP地址均可访问n8n服务器。首次访问需要创建或指定管理员账号密码。
（账户一般为管理员邮箱，如在本例中为admin@example.com）

---

## 二、离线部署

### 2.1 外网下载并传输

```text
[admin@n8n ~]$ docker pull n8nio/n8n:latest
[admin@n8n ~]$ docker save -o n8n_latest.tar n8nio/n8n:latest
[admin@n8n ~]$ scp n8n_latest.tar root@<n8n_server>:/root
[root@diag ~]# ll
8.3.3	加载离线镜像
```

### 2.2 加载离线镜像

```text
[root@n8n ~]# docker load -i /root/n8n_latest.tar
69ea3f5026b2: Loading layer [==================================================>]  150.8MB/150.8MB
5f20ea989587: Loading layer [==================================================>]   5.39MB/5.39MB
604332103181: Loading layer [==================================================>]  3.584kB/3.584kB
37c5f4c9e7a8: Loading layer [==================================================>]  69.07MB/69.07MB
5f70bf18a086: Loading layer [==================================================>]  1.024kB/1.024kB
ef526c4e6600: Loading layer [==================================================>]  799.3MB/799.3MB
92a05c2d8f4c: Loading layer [==================================================>]  11.23MB/11.23MB
a53ed92c626f: Loading layer [==================================================>]  2.048kB/2.048kB
ae21abbc57ff: Loading layer [==================================================>]  3.072kB/3.072kB
df067fd6c252: Loading layer [==================================================>]  5.312MB/5.312MB
64f22576e462: Loading layer [==================================================>]  45.89MB/45.89MB
Loaded image: n8nio/n8n:latest
[root@n8n ~]# docker images | grep n8n
n8nio/n8n                   latest    b4261fdea416   2 days ago    978MB
```

### 2.3 本地部署 n8n

```text
本地部署n8n
```

```bash
[root@n8n ~]# podman run -d --name n8n --restart unless-stopped \
  -p 8080:8080 \
  -v n8n_data:/home/node/.n8n \
  -v /opt/n8n/files:/tmp \
  -e N8N_LISTEN_ADDRESS=0.0.0.0 \
  -e N8N_PORT=8080 \
  -e N8N_PROTOCOL=http \
  -e N8N_HOST=10.66.208.235 \
  -e N8N_SECURE_COOKIE=false \
  n8nio/n8n:latest
```

```text
2c439ebfd20f7366e98ae68e1ad73203d7c9bb0ed048c0bc1d160f5c1ff8a50f
[root@n8n ~]# 
[root@n8n ~]# docker ps -a
CONTAINER ID   IMAGE              COMMAND                  CREATED         STATUS         PORTS                                                   NAMES
2c439ebfd20f   n8nio/n8n:latest   "tini -- /docker-ent…"   5 seconds ago   Up 5 seconds   5678/tcp, 0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp   n8n
[root@n8n ~]# docker logs  -f 2c439ebfd20f
```

### 2.4 验证版本

```text
验证版本
[root@n8n ~]# podman exec -it n8n n8n --version
2.16.1
[root@n8n ~]#
[root@n8n ~]# podman ps --format "{{.Names}} {{.Image}}" | grep n8n
n8n docker.io/n8nio/n8n:latest
[root@n8n ~]# podman exec -it n8n cat /usr/local/lib/node_modules/n8n/package.json | grep version
  "version": "2.16.1",
```
