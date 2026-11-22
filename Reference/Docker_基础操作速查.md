# Docker 基础操作与常见问题速查

## 1. 服务管理

| 命令                       | 作用             |
| -------------------------- | ---------------- |
| `systemctl start docker`   | 启动 Docker      |
| `systemctl stop docker`    | 停止 Docker      |
| `systemctl restart docker` | 重启 Docker      |
| `systemctl status docker`  | 查看 Docker 状态 |
| `systemctl enable docker`  | 设置开机自启     |
| `systemctl disable docker` | 取消开机自启     |

## 2. 镜像管理

```
# 搜索镜像
docker search nginx

# 拉取镜像
docker pull nginx

# 查看本地镜像
docker images

# 删除镜像
docker rmi nginx
```

## 3. 容器管理

```
# 后台运行容器
docker run -d -p 80:80 --name nginx nginx

# 查看运行中的容器
docker ps

# 查看所有容器
docker ps -a

# 进入容器
docker exec -it nginx /bin/bash

# 查看容器日志
docker logs nginx

# 停止容器
docker stop nginx

# 启动已停止的容器
docker start nginx

# 删除容器
docker rm nginx
```

常用参数：

| 参数     | 含义               |
| -------- | ------------------ |
| `-d`     | 后台运行           |
| `-p`     | 端口映射           |
| `--name` | 指定容器名称       |
| `-it`    | 交互式运行         |
| `-v`     | 挂载数据卷         |
| `--rm`   | 容器退出后自动删除 |

## 4. Docker Compose

Vulhub 等漏洞环境通常使用 Docker Compose 管理多个容器。

在包含 `docker-compose.yml` 的项目目录中执行：

```
# 启动服务
docker-compose up -d

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs

# 停止服务
docker-compose stop

# 停止并删除容器、网络
docker-compose down
```
