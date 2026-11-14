# **Kali Linux 代理配置速查**

Kali 中配置代理，主要分为一下几类：

| 代理类型              | 作用范围               | 典型场景                       | 配置方式                        |
| --------------------- | ---------------------- | ------------------------------ | ------------------------------- |
| **环境变量代理**      | 当前 Shell 及其子进程  | curl、wget、Python             | `export HTTP_PROXY=...`         |
| **软件独立代理**      | 指定软件               | APT、Git、pip、npm             | 软件专用配置文件或命令          |
| **服务级代理**        | 后台服务进程           | Docker daemon                  | systemd 环境变量                |
| **程序 / 系统级代理** | 指定应用或系统网络环境 | Firefox、Proxychains、桌面代理 | 应用设置、Proxychains、系统代理 |

## **代理配置检测总表**

| 工具        | 查看配置                                         | 是否独立配置 |
| ----------- | ------------------------------------------------ | ------------ |
| 环境变量    | `env | grep -i proxy`                            | 是           |
| curl        | `env | grep -i proxy`                            | 可指定 `-x`  |
| wget        | `env | grep -i proxy`                            | 可独立配置   |
| APT         | `apt-config dump | grep -i proxy`                | 是           |
| Git         | `git config --global --get-regexp 'http.*proxy'` | 是           |
| Docker      | `systemctl show docker --property=Environment`   | 是           |
| pip         | `pip config list`                                | 是           |
| npm         | `npm config get proxy`                           | 是           |
| Proxychains | `cat /etc/proxychains4.conf`                     | 是           |
| Firefox     | Network Settings                                 | 是           |

## 日常排查命令

```shell
# 1. 查看环境变量代理
env | grep -i proxy

# 2. 查看路由
ip route

# 3. 测试代理服务器连通性
nc -zv 192.168.72.67 7890

# 4. 测试代理访问外网
curl -x http://192.168.72.67:7890 -I https://www.google.com

# 5. 查看 APT 独立代理
apt-config dump | grep -i proxy

# 6. 查看 Git 独立代理
git config --list | grep -i proxy

# 7. 查看 Docker 代理
systemctl show docker --property=Environment
```

下面按实际使用频率整理，默认你的 Windows Clash Verge 监听地址为 `192.168.72.67:7890`

## **1. 基础环境变量代理**

### **配置代理**

```shell
# HTTP 代理
export HTTP_PROXY="http://192.168.72.67:7890"

# HTTPS 代理（代理服务器本身通常仍使用 HTTP 协议）
export HTTPS_PROXY="http://192.168.72.67:7890"

# SOCKS5 代理（如果 Clash 开启了 SOCKS 端口）
export ALL_PROXY="socks5h://192.168.72.67:7890"

# 小写形式，兼容更多程序
export http_proxy="$HTTP_PROXY"
export https_proxy="$HTTPS_PROXY"
export all_proxy="$ALL_PROXY"

# 不走代理的地址
export NO_PROXY="localhost,127.0.0.1,::1,192.168.0.0/16"
export no_proxy="$NO_PROXY"
```

> 注意：HTTP_PROXY、HTTPS_PROXY 和 ALL_PROXY 不一定需要同时配置
>
> 通常 HTTP/HTTPS 两项就够用。`socks5h` 表示 DNS 解析也交给代理端完成

### **检测代理**

```shell
# 查看当前代理环境变量
env | grep -i proxy

# 查看指定变量
echo $HTTP_PROXY
echo $HTTPS_PROXY
echo $ALL_PROXY

# 测试 HTTP/HTTPS 代理
curl -I https://www.google.com

# 强制指定代理测试
curl -x http://192.168.72.67:7890 -I https://www.google.com

# 查看详细连接过程
curl -v https://www.google.com
```

### **取消代理**

```shell
unset HTTP_PROXY HTTPS_PROXY ALL_PROXY NO_PROXY
unset http_proxy https_proxy all_proxy no_proxy
```

### **永久配置**

```shell
# 写入当前用户的 Bash 配置
nano ~/.bashrc

# 添加代理配置
export HTTP_PROXY="http://192.168.72.67:7890"
export HTTPS_PROXY="http://192.168.72.67:7890"
export http_proxy="$HTTP_PROXY"
export https_proxy="$HTTPS_PROXY"

# 立即生效
source ~/.bashrc
```

> 如果你经常使用 root 账户，注意 `~/.bashrc` 指向的是 `/root/.bashrc`，普通用户和 root 的配置互不共享

## **2. APT 代理**

> APT 有自己的代理配置，不要假设它一定会读取 Shell 环境变量

```shell
# 临时使用代理
apt-get -o Acquire::http::Proxy="http://192.168.72.67:7890" \
        -o Acquire::https::Proxy="http://192.168.72.67:7890" update

# 永久配置
cat > /etc/apt/apt.conf.d/99proxy <<'EOF'
Acquire::http::Proxy "http://192.168.72.67:7890";
Acquire::https::Proxy "http://192.168.72.67:7890";
EOF

# 检测 APT 代理配置
apt-config dump | grep -i proxy

# 取消配置
rm /etc/apt/apt.conf.d/99proxy
```

## **3. 常用工具代理**

### **curl / wget**

```shell
# curl：通常直接读取环境变量
curl -I https://www.google.com

# curl：单独指定代理
curl -x http://192.168.72.67:7890 https://www.google.com

# wget：读取代理环境变量
wget https://www.google.com

# wget：查看代理相关环境变量
env | grep -i proxy
```

### **Git**

```shell
# 设置 HTTP 代理
git config --global http.proxy http://192.168.72.67:7890

# 设置 HTTPS 代理
git config --global https.proxy http://192.168.72.67:7890

# 查看代理配置
git config --global --get-regexp 'http.*proxy'

# 取消代理
git config --global --unset http.proxy
git config --global --unset https.proxy

# 测试 GitHub 连接
git ls-remote https://github.com/git/git.git
```

>  注意：Git 的独立配置可能覆盖环境变量

### **Docker**

Docker 分为两种情况：

- Docker CLI：执行 `docker pull` 时，实际下载工作由 Docker daemon 负责
- Docker daemon：需要单独配置代理

```shell
# 创建 Docker 代理配置目录
mkdir -p /etc/systemd/system/docker.service.d

# 配置 Docker daemon 代理
cat > /etc/systemd/system/docker.service.d/proxy.conf <<'EOF'
[Service]
Environment="HTTP_PROXY=http://192.168.72.67:7890"
Environment="HTTPS_PROXY=http://192.168.72.67:7890"
Environment="NO_PROXY=localhost,127.0.0.1"
EOF

# 重新加载 systemd
systemctl daemon-reload

# 重启 Docker
systemctl restart docker

# 检测 Docker 服务环境变量
systemctl show docker --property=Environment

# 测试镜像拉取
docker pull hello-world
```

> 注意：这里配置的是 Docker daemon，不是当前终端的环境变量

### **Python pip**

```shell
# 临时指定代理
pip install requests --proxy http://192.168.72.67:7890

# 永久配置代理
pip config set global.proxy http://192.168.72.67:7890

# 查看配置
pip config list

# 取消代理
pip config unset global.proxy
```

### **Node.js npm**

```shell
# 设置代理
npm config set proxy http://192.168.72.67:7890
npm config set https-proxy http://192.168.72.67:7890

# 查看代理
npm config get proxy
npm config get https-proxy

# 取消代理
npm config delete proxy
npm config delete https-proxy

# 测试
npm ping
```

## **4. 其他常用代理工具**

### **Proxychains（强制程序走代理）**

> 适合一些不读取环境变量的命令行程序，例如某些扫描工具

```
# 安装
apt install proxychains4

# 编辑配置
nano /etc/proxychains4.conf

# 在文件末尾配置代理（示例）
# socks5 192.168.72.67 7891

# 通过代理执行程序
proxychains4 curl https://www.google.com

# 通过代理执行 Git
proxychains4 git clone https://github.com/git/git.git
```

> 注意：这里假设 Clash 的 SOCKS5 端口是 `7891`，需要根据实际配置填写
>
> `Proxychains` 不是所有程序都兼容

### **浏览器**

以 Firefox 为例：

1. 打开 Settings
2. 搜索 Network Settings
3. 选择 Manual proxy configuration
4. 填写 HTTP Proxy 和端口 `7890`
5. 勾选 HTTPS 相关代理配置

> 浏览器有自己的代理设置，不一定跟随终端环境变量

## **日常排查命令**

下面这组建议保留，遇到 Kali 无法联网时按顺序检查

```
# 1. 查看环境变量代理
env | grep -i proxy

# 2. 查看路由
ip route

# 3. 测试代理服务器连通性
nc -zv 192.168.72.67 7890

# 4. 测试代理访问外网
curl -x http://192.168.72.67:7890 -I https://www.google.com

# 5. 查看 APT 独立代理
apt-config dump | grep -i proxy

# 6. 查看 Git 独立代理
git config --list | grep -i proxy

# 7. 查看 Docker 代理
systemctl show docker --property=Environment
```



# 拓展阅读

## 关于 `NO_PROXY`

`NO_PROXY` 的作用是：**指定哪些目标地址不经过代理，直接连接。**

正常情况下，你配置了：

```shell
export HTTP_PROXY="http://192.168.72.67:7890"
export HTTPS_PROXY="http://192.168.72.67:7890"
```

那么支持环境变量代理的程序访问外部网站时，会尝试通过代理服务器转发

但有些地址不需要代理，例如本机服务、局域网靶机，这时就用 `NO_PROXY` 排除

## **1. 逐个解释**

```shell
export NO_PROXY="localhost,127.0.0.1,::1,192.168.0.0/16"
```

| 地址             | 含义                              | 示例                    |
| ---------------- | --------------------------------- | ----------------------- |
| `localhost`      | 本机主机名                        | `http://localhost:8080` |
| `127.0.0.1`      | IPv4 本机回环地址                 | `http://127.0.0.1:8080` |
| `::1`            | IPv6 本机回环地址                 | `http://[::1]:8080`     |
| `192.168.0.0/16` | 192.168.0.0～192.168.255.255 网段 | `192.168.12.135:7001`   |

最后一个尤其值得注意：

`192.168.0.0/16` 是 CIDR 网段表示法，表示整个 `192.168.x.x` 私有地址范围

例如：

- `192.168.12.135`：不走代理
- `192.168.72.67`：不走代理
- `192.168.1.100`：不走代理

## **2. 在 Kali 中有什么用？**

假设你正在使用 Vulhub

你的环境：

- Kali：`192.168.12.135`
- Windows Clash：`192.168.72.67:7890`
- WebLogic 靶机：`192.168.12.135:7001`

当你执行：

```shell
curl http://192.168.12.135:7001
```

如果没有设置 `NO_PROXY`，curl 可能会尝试通过代理访问靶机

设置之后：

```shell
export NO_PROXY="localhost,127.0.0.1,::1,192.168.0.0/16"
```

访问 `192.168.12.135:7001` 就会直接连接，不经过代理

## **3. 如何检测？**

```shell
# 查看排除列表
echo $NO_PROXY

# 强制指定代理，观察是否绕过
curl -v https://192.168.12.135:7001
```

注意：`NO_PROXY` 的网段匹配支持情况取决于具体程序和版本，不能假设所有软件都支持 CIDR

**简单记忆：**

- `HTTP_PROXY`：哪些请求可以通过 HTTP 代理
- `HTTPS_PROXY`：哪些 HTTPS 请求可以通过代理
- `ALL_PROXY`：通用代理后备配置
- `NO_PROXY`：哪些目标不需要代理，直接连接