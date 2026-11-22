## 一、WSL Kali 重装与初始化速查

### 1. 卸载与重装（Windows 终端）

```PowerShell
wsl -l -v                        # 查看现有子系统名称与版本
wsl --unregister kali-linux      # 彻底注销并删除旧 Kali（清空数据）
wsl --install -d kali-linux      # 拉取并安装全新 Kali
```

### 2. Root 与权限设置（Kali 终端）

```Bash
sudo passwd root                 # 设置 root 密码
su - root                        # 切换到 root 用户
sudo usermod -aG sudo <用户名>    # 赋予普通用户 sudo 管理员权限
```

### 3. 换国内源与配代理（先换源再装包，避免报错）

```Bash
# 强制覆盖为阿里云源（彻底绕开官方 CDN 抽风）
echo "deb https://mirrors.aliyun.com/kali kali-rolling main non-free contrib" | sudo tee /etc/apt/sources.list

# 终端全局代理（root 默认是 Bash，必须写进 ~/.bashrc）
echo 'export http_proxy="http://192.168.163.78:7890"' >> ~/.bashrc
echo 'export https_proxy="http://192.168.163.78:7890"' >> ~/.bashrc
echo 'export all_proxy="socks5://192.168.163.78:7890"' >> ~/.bashrc
source ~/.bashrc

# 测试外网代理是否打通
curl -I https://www.google.com
```

> 若 APT 本身也要走代理（通常换了阿里云源就不需要），再补执行：
>
> ```shell
> echo -e "Acquire::http::Proxy \"[http://192.168.163.78:7890](http://192.168.163.78:7890)\";\nAcquire::https::Proxy \"[http://192.168.163.78:7890](http://192.168.163.78:7890)\";" | sudo tee /etc/apt/apt.conf.d/proxy.conf
> ```

### 4. 系统更新与核心工具/Docker 一键安装

```Bash
# 同步源并升级底层依赖、清理无用包
sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y

# 补齐常用渗透工具、字典集与 Docker 引擎
sudo apt install -y nmap sqlmap masscan gobuster dirsearch metasploit-framework \
hydra hashcat john binwalk proxychains4 tmux git curl wget jq tree \
wordlists seclists docker.io docker-compose

# 启动 Docker 服务并验证（WSL 重启后需重新 start）
sudo service docker start && docker --version
```

### 5. Proxychains 与 Pipx 工具链

```Bash
# 修改 Proxychains 出口，让 nmap 等不吃环境变量的工具强制走宿主机代理
sudo sed -i 's/^socks4 \+127\.0\.0\.1 \+9050/socks5  192.168.163.78 7890/' /etc/proxychains4.conf

# 安装 Pipx（绕过 Kali 的 PEP 668 限制，隔离安装 Python 安全脚本）
sudo apt install -y python3-pip pipx && pipx ensurepath && source ~/.bashrc
```

### 6. Rockyou 字典解压与快捷路径

```Bash
# 解压千万级弱口令字典为纯文本（方便爆破工具高速读取）
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz

# 在家目录建软链接，实战时直接敲 ~/rockyou.txt 替代冗长路径
ln -s /usr/share/wordlists/rockyou.txt ~/rockyou.txt
```

### 7. MSF 数据库初始化与启动

```Bash
sudo service postgresql start               # 启动 PostgreSQL 服务（WSL 重启后需重新执行）
sudo msfdb init                             # 初始化 MSF 库表（仅首次安装执行一次）
msfconsole -q -x "db_status; exit"          # 静默验证连接（回显 Connected to msf 即成功）
```

## 二、解答笔记疑问：MSF 数据库怎么用？

连上数据库后，进 `msfconsole` 直接用下面这几个高频命令，不用再手动记笔记：

```Bash
workspace -a box01            # 新建并切换到独立工作区（建议一个靶机建一个，数据隔离）
workspace                     # 查看所有工作区（带 * 号的是当前所在）
db_nmap -sV -Pn <目标IP>      # 在 MSF 里直接调 Nmap 扫描，结果自动写入数据库
hosts                         # 查看已入库的存活主机列表
services                      # 查看已入库的开放端口与服务版本
vulns                         # 查看记录到的漏洞信息
creds                         # 查看抓取或爆破出来的账号密码凭证
loot                          # 查看收集到的敏感文件与哈希
```

## 三、本次踩坑与排错速查

| **遇到的问题**     | **报错特征**                                                 | **根本原因**                                             | **解决方案**                                                 |
| ------------------ | ------------------------------------------------------------ | -------------------------------------------------------- | ------------------------------------------------------------ |
| **Shell 配置冲突** | `source ~/.zshrc` 刷屏报 `setopt`/`bindkey: command not found` | 切到 `root` 后默认是 **Bash**，无法解析 **Zsh** 专属语法 | 把代理环境变量改写入 `~/.bashrc`，执行 `source ~/.bashrc`    |
| **官方源拦截代理** | 安装 `masscan` 报 `403 Forbidden`                            | Kali 官方 CDN 识别到代理 IP 的批量拉取并触发拦截         | 换阿里云镜像源，再跑 `sudo apt update --fix-missing` 补装剩余包 |
| **镜像节点故障**   | 安装 Docker 时 `tini-static` 报 `502 Bad Gateway`            | 官方 CDN 随机分配的新加坡节点宕机或与代理断连            | 用 `echo` 将阿里云源直接覆写进 `/etc/apt/sources.list`，再加 `--fix-missing` 完成安装 |

## 四、MSF 数据库的日常使用

在 WSL 里，MSF 数据库的日常使用可以浓缩为一个固定套路：

**启动服务 -> 建工作区 -> 存数据 -> 查数据/联动攻击 -> 导出备份**。

### 1. 每次重启 WSL 后的前置动作

WSL 默认不自启后台服务，每次重新打开电脑进 Kali 后，先唤醒数据库再进 MSF：

```Bash
sudo service postgresql start
msfconsole -q
```

进控制台后敲一下 `db_status`，看到 `Connected to msf` 就说明在线

### 2. 打一台新靶机的标准流程

**第一步：建独立工作区（防串数据）** 

打新靶机或做新项目前，先隔离出一个专属空间：

```Bash
workspace -a target_01     # 新建并自动切换到 target_01
workspace                  # 查看所有工作区（带 * 号的是当前所在）
workspace default          # 切回默认工作区
workspace -d target_01     # 打完不需要了，删掉整个工作区
```

**第二步：把扫描数据灌进库里（两种方式）**

- **直接在 MSF 里扫：** 语法和普通 Nmap 一样，结果会自动入库。

  ```Bash
  db_nmap -sV -sC -Pn 10.10.10.10
  ```

- **导入外部 Nmap 结果（更常用）：** 如果你在外面用 Nmap 扫完保存了 XML（`nmap -oX scan.xml ...`），直接导进来即可。

  ```Bash
  db_import /path/to/scan.xml
  ```

**第三步：随时翻看资产与战果** 不用再往上翻几十页终端记录，直接查表：

```Bash
hosts                      # 看存活主机、IP、操作系统
services                   # 看所有开放端口、协议和服务版本
services -p 80,443         # 只筛选 80 和 443 端口
vulns                      # 看扫描或利用过程中记录的漏洞
creds                      # 看爆破或抓取到的账号、密码、Hash
loot                       # 看拿到的敏感文件（如 /etc/shadow、配置文件等）
```

### 3. 最好用的进阶技巧：数据库联动模块

数据库不只是记事本，它最爽的地方是**直接把查出来的结果塞进当前模块的参数里**（`-R` 参数）：

```Bash
# 比如你选了一个 SMB 扫描模块
use auxiliary/scanner/smb/smb_version

# 直接把数据库里所有开了 445 端口的 IP 自动填入 RHOSTS
services -p 445 -R

# 确认一下 RHOSTS 已经自动填好了，直接开跑
show options
run
```

打多台机器的内网渗透时，这个技巧能省掉大量手动复制粘贴 IP 的时间

### 4. 打完收工：导出数据写报告

需要归档或把数据发给队友同步进度时，把当前工作区打包导出：

```Bash
db_export -f xml ~/target_01_report.xml
```

队友拿到这个 XML 文件后，在自己的 MSF 里执行 `db_import` 就能直接复现你的全部资产和凭证记录
