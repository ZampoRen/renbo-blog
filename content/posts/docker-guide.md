---
title: "Docker容器化技术入门与实战"
date: 2024-02-20T10:15:00+08:00
draft: false
author: "博主"
description: "从零开始学习Docker容器技术，包括基本概念、常用命令、Dockerfile编写、容器编排等核心内容。"
tags: ["Docker", "容器化", "DevOps", "运维"]
categories: ["技术"]
---

Docker改变了现代软件部署和运维的方式。今天我们来深入了解这项革命性的容器技术。

## 🐳 什么是Docker

### 容器 vs 虚拟机

| 特性 | 容器 | 虚拟机 |
|------|------|--------|
| **资源占用** | 轻量级 | 重量级 |
| **启动速度** | 秒级 | 分钟级 |
| **性能损耗** | 几乎无 | 10-30% |
| **隔离程度** | 进程级 | 硬件级 |
| **镜像大小** | MB级 | GB级 |

### Docker核心概念

```
Docker架构：
├── Docker Client（客户端）
├── Docker Daemon（守护进程）
├── Docker Images（镜像）
├── Docker Containers（容器）
├── Docker Registry（仓库）
└── Docker Network（网络）
```

## 🚀 Docker安装与配置

### 在不同系统上安装

#### macOS
```bash
# 使用Homebrew安装
brew install docker
brew install docker-compose

# 或下载Docker Desktop
# https://www.docker.com/products/docker-desktop
```

#### Ubuntu
```bash
# 更新包索引
sudo apt update

# 安装依赖
sudo apt install apt-transport-https ca-certificates curl software-properties-common

# 添加Docker官方GPG密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# 添加Docker仓库
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# 安装Docker CE
sudo apt update
sudo apt install docker-ce

# 启动Docker服务
sudo systemctl start docker
sudo systemctl enable docker
```

#### CentOS/RHEL
```bash
# 安装依赖
sudo yum install -y yum-utils

# 添加Docker仓库
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 安装Docker
sudo yum install docker-ce docker-ce-cli containerd.io

# 启动服务
sudo systemctl start docker
sudo systemctl enable docker
```

### 配置Docker

#### 将用户添加到docker组
```bash
sudo usermod -aG docker $USER
# 注销并重新登录生效
```

#### 配置镜像加速器
```json
# /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
```

## 📦 镜像管理

### 基本镜像操作

```bash
# 搜索镜像
docker search nginx

# 拉取镜像
docker pull nginx:latest
docker pull node:16-alpine

# 查看本地镜像
docker images
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# 删除镜像
docker rmi nginx:latest
docker rmi $(docker images -q)  # 删除所有镜像

# 镜像详细信息
docker inspect nginx:latest

# 镜像历史
docker history nginx:latest
```

### 构建自定义镜像

#### 编写Dockerfile
```dockerfile
# 基础镜像
FROM node:16-alpine

# 维护者信息
LABEL maintainer="your-email@example.com"
LABEL version="1.0"
LABEL description="Node.js application"

# 设置工作目录
WORKDIR /app

# 复制package文件
COPY package*.json ./

# 安装依赖
RUN npm ci --only=production && npm cache clean --force

# 复制应用代码
COPY . .

# 创建非root用户
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001
USER nextjs

# 暴露端口
EXPOSE 3000

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# 启动命令
CMD ["npm", "start"]
```

#### 多阶段构建
```dockerfile
# 构建阶段
FROM node:16-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# 生产阶段
FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### 构建镜像
```bash
# 构建镜像
docker build -t my-app:v1.0 .
docker build -t my-app:latest --no-cache .

# 使用构建参数
docker build --build-arg NODE_VERSION=16 -t my-app .

# 查看构建过程
docker build -t my-app --progress=plain .
```

## 🏃 容器运行管理

### 基本容器操作

```bash
# 运行容器
docker run hello-world
docker run -d nginx  # 后台运行
docker run -it ubuntu bash  # 交互模式

# 端口映射
docker run -d -p 8080:80 nginx
docker run -d -p 127.0.0.1:8080:80 nginx

# 挂载目录
docker run -d -v /host/path:/container/path nginx
docker run -d -v my-volume:/app/data nginx

# 环境变量
docker run -d -e NODE_ENV=production -e PORT=3000 my-app

# 资源限制
docker run -d --memory=512m --cpus=0.5 nginx
```

### 容器生命周期管理

```bash
# 查看运行中的容器
docker ps
docker ps -a  # 包括已停止的容器

# 启动/停止容器
docker start container_id
docker stop container_id
docker restart container_id

# 暂停/恢复容器
docker pause container_id
docker unpause container_id

# 删除容器
docker rm container_id
docker rm -f container_id  # 强制删除运行中的容器
docker container prune  # 删除所有停止的容器
```

### 容器交互

```bash
# 进入运行中的容器
docker exec -it container_id bash
docker exec -it container_id sh

# 查看容器日志
docker logs container_id
docker logs -f container_id  # 实时查看
docker logs --tail 100 container_id  # 查看最后100行

# 容器资源使用情况
docker stats
docker stats container_id

# 容器详细信息
docker inspect container_id
```

## 🔧 Docker Compose

### 什么是Docker Compose
Docker Compose是用于定义和运行多容器Docker应用程序的工具。

### 编写docker-compose.yml

#### 基础示例
```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
    depends_on:
      - db
    volumes:
      - .:/app
      - node_modules:/app/node_modules

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: myapp
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppassword
    volumes:
      - db_data:/var/lib/mysql
    ports:
      - "3306:3306"

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

volumes:
  db_data:
  node_modules:
```

#### 复杂示例（LNMP架构）
```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./www:/var/www/html
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - php
    networks:
      - app-network

  php:
    build:
      context: ./php
      dockerfile: Dockerfile
    volumes:
      - ./www:/var/www/html
    networks:
      - app-network
    depends_on:
      - mysql

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "3306:3306"
    networks:
      - app-network

  phpmyadmin:
    image: phpmyadmin:latest
    environment:
      PMA_HOST: mysql
      PMA_PORT: 3306
    ports:
      - "8080:80"
    depends_on:
      - mysql
    networks:
      - app-network

volumes:
  mysql_data:

networks:
  app-network:
    driver: bridge
```

### Compose命令

```bash
# 启动服务
docker-compose up
docker-compose up -d  # 后台启动
docker-compose up --build  # 重新构建

# 停止服务
docker-compose stop
docker-compose down  # 停止并删除容器
docker-compose down -v  # 同时删除数据卷

# 查看服务状态
docker-compose ps
docker-compose logs web  # 查看特定服务日志

# 扩展服务
docker-compose up --scale web=3

# 执行命令
docker-compose exec web bash
docker-compose run web npm install
```

## 🌐 网络管理

### Docker网络类型

```bash
# 查看网络
docker network ls

# 创建自定义网络
docker network create my-network
docker network create --driver bridge --subnet=172.20.0.0/16 my-custom-network

# 连接容器到网络
docker run -d --network my-network nginx
docker network connect my-network container_id

# 网络详细信息
docker network inspect my-network
```

### 网络配置示例

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    image: nginx
    networks:
      - frontend-network
      - backend-network

  api:
    image: node:alpine
    networks:
      - backend-network
      - database-network

  database:
    image: postgres
    networks:
      - database-network

networks:
  frontend-network:
    driver: bridge
  backend-network:
    driver: bridge
  database-network:
    driver: bridge
    internal: true  # 内部网络，不能访问外网
```

## 💾 数据卷管理

### 数据卷类型

```bash
# 命名卷
docker volume create my-volume
docker volume ls
docker volume inspect my-volume

# 使用数据卷
docker run -d -v my-volume:/app/data nginx

# 绑定挂载
docker run -d -v /host/path:/container/path nginx

# 临时文件系统
docker run -d --tmpfs /app/temp nginx
```

### 数据备份与恢复

```bash
# 备份数据卷
docker run --rm -v my-volume:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz -C /data .

# 恢复数据卷
docker run --rm -v my-volume:/data -v $(pwd):/backup alpine tar xzf /backup/backup.tar.gz -C /data

# 数据卷之间复制
docker run --rm -v old-volume:/from -v new-volume:/to alpine sh -c "cp -a /from/. /to/"
```

## 🔍 监控和日志

### 容器监控

```bash
# 实时监控
docker stats
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# 系统信息
docker system df  # 磁盘使用情况
docker system info  # 系统信息
docker system prune  # 清理未使用的资源
```

### 日志管理

```bash
# 查看日志
docker logs container_id
docker logs -f --tail 100 container_id

# 日志驱动配置
docker run -d --log-driver=json-file --log-opt max-size=10m --log-opt max-file=3 nginx
```

#### 集中化日志配置
```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    image: my-app
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  nginx:
    image: nginx
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://192.168.0.42:123"
```

## 🛡️ 安全最佳实践

### 镜像安全

```dockerfile
# 使用官方基础镜像
FROM node:16-alpine

# 不要以root用户运行
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001
USER nextjs

# 最小权限原则
COPY --chown=nextjs:nodejs . .

# 扫描漏洞
RUN npm audit --audit-level moderate
```

### 运行时安全

```bash
# 只读根文件系统
docker run --read-only --tmpfs /tmp nginx

# 限制能力
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# 使用非特权用户
docker run -u 1001:1001 nginx

# 资源限制
docker run --memory=512m --cpus=0.5 --ulimit nofile=1024:1024 nginx
```

## 🚀 实际应用场景

### 1. 开发环境统一

```bash
# 开发环境脚本
#!/bin/bash
echo "启动开发环境..."

# 启动数据库
docker run -d --name dev-db \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=app_dev \
  -p 3306:3306 \
  mysql:8.0

# 启动Redis
docker run -d --name dev-redis \
  -p 6379:6379 \
  redis:alpine

# 启动应用
docker run -d --name dev-app \
  -p 3000:3000 \
  -e NODE_ENV=development \
  -v $(pwd):/app \
  node:16-alpine npm run dev

echo "开发环境启动完成！"
```

### 2. 微服务部署

```yaml
# microservices-stack.yml
version: '3.8'

services:
  api-gateway:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf

  user-service:
    image: user-service:latest
    environment:
      - DB_HOST=postgres
      - REDIS_HOST=redis

  order-service:
    image: order-service:latest
    environment:
      - DB_HOST=postgres
      - RABBITMQ_HOST=rabbitmq

  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: microservices
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:alpine

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"

volumes:
  postgres_data:
```

### 3. CI/CD集成

```yaml
# .github/workflows/docker.yml
name: Docker Build and Deploy

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Build Docker image
      run: docker build -t my-app:${{ github.sha }} .
    
    - name: Run tests
      run: docker run --rm my-app:${{ github.sha }} npm test
    
    - name: Push to registry
      run: |
        echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
        docker push my-app:${{ github.sha }}
    
    - name: Deploy to production
      run: |
        docker-compose -f docker-compose.prod.yml pull
        docker-compose -f docker-compose.prod.yml up -d
```

## 📚 学习资源和工具

### 官方资源
- **Docker官方文档**：https://docs.docker.com/
- **Docker Hub**：https://hub.docker.com/
- **Play with Docker**：https://labs.play-with-docker.com/

### 实用工具
```bash
# Docker图形化管理工具
docker run -d -p 9000:9000 --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce

# 容器监控
docker run -d --name cAdvisor \
  -p 8080:8080 \
  -v /:/rootfs:ro \
  -v /var/run:/var/run:ro \
  -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro \
  google/cadvisor:latest
```

## 💡 总结

Docker容器化技术的核心优势：

1. **环境一致性**：开发、测试、生产环境完全一致
2. **快速部署**：秒级启动，版本回滚方便
3. **资源高效**：比虚拟机更轻量，资源利用率高
4. **易于扩展**：水平扩展简单，支持微服务架构
5. **生态丰富**：Docker Hub提供大量现成镜像

**学习建议**：
- 🎯 **从基础开始**：掌握基本概念和命令
- 🛠️ **多实践**：自己动手构建镜像和容器
- 📚 **深入理解**：学习Docker原理和最佳实践
- 🔄 **结合CI/CD**：将Docker融入开发流程
- 🌐 **关注生态**：了解Kubernetes等容器编排工具

Docker不仅仅是一个工具，它代表了一种新的软件交付理念。掌握Docker，让你的应用部署更加简单、高效、可靠！

---

*你在使用Docker过程中遇到了哪些问题？欢迎在评论区分享经验！*