# 前后端 Docker 部署模板

> 不含业务代码,只教你把打包好的前后端产物用 Docker 跑起来。

一套基于 Docker Compose 的前后端一键部署模板,适合 **Spring Boot 后端 + 前端 SPA(Vue/React 等)+ Nginx** 的项目。

本仓库的目标是帮你理解「**打包好的产物如何在服务器上用 Docker 跑起来**」这件事,所以它**不包含业务源码**,只包含部署相关的配置文件。你只需要把自己项目的产物(前端 `dist`、后端 `jar`)放进对应目录,改几处配置,就能部署。

> 适合人群:第一次用 Docker 部署前后端项目的同学。如果你已经很熟,可以直接看[目录结构](#目录结构)和[配置说明](#配置说明)。

## ✨ 这个模板能给你什么

- 🚀 **一键部署**:`docker compose up -d` 跑起前后端
- 🔒 **HTTPS 开箱即用**:80 自动跳转 443,配置好反向代理
- 📦 **镜像自包含**:配置打进镜像,版本清晰,回滚方便
- 🛡️ **安全默认**:后端不对外暴露,只走 Nginx 代理
- 📖 **详细中文教程**:每个配置都讲清「为什么」,新手友好
- 🎯 **不含业务代码**:纯部署模板,放上你的产物即可

---

## 架构总览

```
                    互联网
                      │
                      ▼
            ┌──────────────────┐
            │   Nginx (前端容器) │  80 → 强制跳转 443
            │   80 / 443 端口    │  443 提供页面 + 反向代理
            └────────┬─────────┘
                     │ /api/ 转发(容器内网，http)
                     ▼
            ┌──────────────────┐
            │ Spring Boot(后端) │  仅容器内网暴露 8080
            │      8080         │  不对宿主机开放
            └──────────────────┘

         两个容器通过自定义 bridge 网络互相通信
```

- **前端容器(Nginx)**:对外提供 80/443,负责静态页面 + 把 `/api/` 请求反向代理给后端。
- **后端容器(Spring Boot)**:只在 Docker 内网暴露 8080,不直接对外,所有外部请求都经过 Nginx。
- 浏览器到 Nginx 走 **HTTPS(443)**;Nginx 到后端走容器内网的 HTTP,这段不出宿主机,是安全的。

---

## 前置要求

服务器(以 Linux 为例)需要安装:

- Docker(20.10+)
- Docker Compose(v2,命令是 `docker compose`,注意没有中划线)

检查是否装好:

```bash
docker -v
docker compose version
```

没装的话,可参考 Docker 官方文档安装,这里不展开。

---

## 目录结构

把整个目录放到服务器的 `/opt` 下(也可以是别的路径,下文以 `/opt` 为例):

```
/opt
├── docker-compose.yml          # 编排文件：定义前后端两个服务
├── backend/                    # 后端
│   ├── Dockerfile              # 后端镜像构建脚本
│   └── xxx.jar                 # 【你来放】后端打包出的 jar
└── frontend/                   # 前端
    ├── Dockerfile              # 前端镜像构建脚本
    ├── default.conf            # Nginx 配置（含反向代理、SSL）
    ├── dist/                   # 【你来放】前端打包出的静态文件
    └── ssl/                    # 【你来放】HTTPS 证书
        ├── fullchain.pem
        └── certkey.pem
```

标注【你来放】的是需要你自己提供的产物,仓库里不包含。其余配置文件可以直接用,改几处即可。

---

## 配置说明

部署前,有几处和你的项目 / 服务器强相关,**必须替换成你自己的值**。

### 1. 后端 Dockerfile（`backend/Dockerfile`)

```dockerfile
FROM <你的JDK基础镜像>
WORKDIR /opt
COPY your-app.jar app.jar
RUN mkdir -p /opt/logs
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "exec java -jar app.jar"]
```

要改的地方:

- `FROM`:换成你自己的 JDK 基础镜像。可以用官方的 `eclipse-temurin:8-jre`(JDK8)或 `eclipse-temurin:17-jre`(JDK17),也可以用你自己镜像仓库里的。
- `COPY your-app.jar app.jar`:把 `your-app.jar` 换成你 jar 包的真实文件名。

> **为什么用 `exec java -jar`?**
> 加 `exec` 是为了让 Java 进程成为容器的 1 号进程(PID 1)。这样 `docker stop` 发出的停止信号能直接被 Java 收到,触发 Spring Boot 的优雅停机(处理完手头请求再退出、正常释放数据库连接),而不是被强制杀掉。

### 2. 前端 Nginx 配置（`frontend/default.conf`)

```nginx
upstream backend {
    server backend:8080;   # 后端容器名:端口，一般不用改
}

# 80 端口：强制跳转到 https
server {
    listen 80;
    server_name your-domain.com;          # ← 改成你的域名
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name your-domain.com;          # ← 改成你的域名
    client_max_body_size 1024m;           # 上传大文件时按需调整

    ssl_certificate /opt/nginx/ssl/fullchain.pem;     # 证书路径，对应 ssl/ 目录
    ssl_certificate_key /opt/nginx/ssl/certkey.pem;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;  # SPA 路由：刷新不 404
    }

    location /api/ {
        proxy_pass http://backend;         # 转发给后端，内网 http
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

要改的地方:

- `server_name`:两处都改成你的域名。
- `ssl_certificate` / `ssl_certificate_key`:文件名要和你放进 `ssl/` 目录的证书一致。
- `location /api/`:这里假设你后端接口都以 `/api/` 开头。如果你的接口前缀不同(比如没有前缀),需要相应调整。

> **关于 HTTPS 证书**:你需要自己申请(免费的可以用 Let's Encrypt,或云厂商提供的免费证书),拿到 `fullchain.pem` 和 `certkey.pem`(或对应的 `.crt`/`.key`,改下配置里的文件名即可)放进 `ssl/` 目录。
>
> **不想用 HTTPS / 还没有域名怎么办?** 如果只是本地或内网测试,可以把 `default.conf` 里 443 那段删掉,把 80 那段从「跳转」改回「提供服务」(即把后面 443 段的 `location` 内容搬到 80 段里)。生产环境强烈建议用 HTTPS。

### 3. 编排文件（`docker-compose.yml`)

```yaml
services:
  backend:
    image: backend:v1.0.0             # ← 镜像名:版本号，每次发布建议手动 +1
    container_name: backend
    build:
      context: ./backend
    expose:
      - "8080"                         # 只在内网暴露，不映射到宿主机
    volumes:
      - /opt/app-data:/opt/data        # ← 业务数据/上传文件，按需修改
      - /opt/backend/logs:/opt/logs    # 日志挂载到宿主机，方便查看
    environment:
      - SPRING_PROFILES_ACTIVE=prod    # 激活的 Spring 配置文件
      - TZ=Asia/Shanghai               # 时区
    restart: unless-stopped            # 崩溃或重启后自动拉起
    networks:
      - app-network

  frontend:
    image: frontend:v1.0.0
    container_name: frontend
    build:
      context: ./frontend
    ports:
      - "80:80"
      - "443:443"
    restart: unless-stopped
    depends_on:
      - backend
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

要改的地方:

- `image`:镜像名和版本号自定。**建议每次发布手动把版本号 +1**(如 `v1.0.0` → `v1.0.1`),这样镜像有清晰的历史,出问题能切回旧版本。
- `volumes`:把需要持久化的目录(用户上传文件、数据等)挂载到宿主机。**容器删了重建,挂载在宿主机的数据不会丢**;反之,没挂载的数据随容器销毁而消失。
- `environment`:按你后端实际需要的环境变量增减。

> **挂载(volumes)的核心原则**:**状态数据**(用户上传、日志、数据库文件)挂载到宿主机;**代码和配置**(jar、dist、nginx 配置)打进镜像。这样镜像是「不可变」的,换版本不影响数据。

---

## 部署步骤

### 第一次部署

1. **拉取模板**:

   ```bash
   git clone https://github.com/gem921/fullstack-docker-template.git
   cd fullstack-docker-template
   ```

2. **准备产物**:本地打包好前端和后端。

   ```bash
   # 前端（在前端项目目录）
   npm run build          # 生成 dist/

   # 后端（在后端项目目录）
   mvn clean package      # 生成 target/xxx.jar
   ```

3. **放置文件**:把 `dist/` 放进 `frontend/`,把 `jar` 放进 `backend/`,把证书放进 `frontend/ssl/`。

4. **上传到服务器**:把整个目录传到服务器(可用 `scp`、`rsync` 或任意 SFTP 工具),下文以放在 `/opt` 为例。

5. **构建并启动**:

   ```bash
   cd /opt
   docker compose build       # 构建前后端镜像
   docker compose up -d        # 后台启动
   ```

6. **验证**:

   ```bash
   docker compose ps           # 看两个容器是否都是 running
   docker compose logs -f      # 看启动日志，Ctrl+C 退出
   ```

   浏览器访问你的域名,应该能看到前端页面;接口请求 `/api/...` 应正常返回。

### 后续更新发布

改了代码、要发新版本时:

1. 重新打包前端 / 后端,替换掉 `dist/` 和 `jar`(改了 nginx 配置或证书也一样)。
2. 把 `docker-compose.yml` 里的 `image` 版本号 **+1**。
3. 上传覆盖到服务器。
4. 重新构建并启动:

   ```bash
   cd /opt
   docker compose build
   docker compose up -d        # 会用新镜像重建有变化的容器
   ```

> **注意**:本模板把前端的 `dist`、`ssl`、nginx 配置都**打进了镜像**。所以只要改了这些文件,就必须重新 `docker compose build`,光 `up -d` 不会生效。这是「镜像自包含」带来的代价,换来的是镜像不可变、版本清晰、回滚方便。

---

## 常用命令速查

```bash
docker compose ps               # 查看服务状态
docker compose logs -f          # 实时查看所有日志
docker compose logs -f backend  # 只看后端日志
docker compose restart          # 重启所有服务
docker compose down             # 停止并删除容器（不删镜像、不删挂载数据）
docker compose up -d --build    # 构建 + 启动一步到位
docker images                   # 查看本地镜像列表
```

---

## 常见问题(FAQ)

**Q：访问页面 502 Bad Gateway?**
后端没起来或还在启动中。`docker compose logs -f backend` 看后端日志。Nginx 的 `upstream` 连不上后端时就会 502。

**Q：前端页面能开,但接口请求失败 / 404?**
检查三点:① 后端容器是否正常运行;② `default.conf` 里 `location /api/` 的前缀和你后端接口前缀是否一致;③ 后端接口本身是否正常(可在容器内 `curl` 测)。

**Q：刷新前端页面就 404?**
确认 `default.conf` 里有 `try_files $uri $uri/ /index.html;`,这是 SPA 单页应用刷新不丢路由的关键。

**Q：改了 nginx 配置 / 前端代码,`up -d` 后没变化?**
因为配置打进了镜像,必须先 `docker compose build` 重新构建,再 `up -d`。

**Q：HTTPS 证书过期了怎么换?**
替换 `ssl/` 目录下的证书文件,然后重新 `docker compose build && docker compose up -d`。

**Q：怎么回滚到上一个版本?**
如果旧版本镜像还在(`docker images` 能看到),把 `docker-compose.yml` 的 `image` 版本号改回旧的,然后 `docker compose up -d` 即可。这就是建议每次发布手动 +1 版本号的意义。

---

## 说明

- 本模板中的镜像名、目录名(`backend` / `frontend`)仅为示例,你可以全部替换成自己项目的名字。
- 配置中涉及域名、证书、镜像仓库地址的部分均为占位符,请替换为你自己的真实值。**不要把私钥证书、含密码的配置文件提交到公开仓库。**

欢迎根据自己的项目调整。如果这个模板帮到了你,欢迎给 [gem921/fullstack-docker-template](https://github.com/gem921/fullstack-docker-template) 点个 Star 支持一下 🌟

