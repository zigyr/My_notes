# Kali - Wordlists - RockYou 与 SecLists 速查手册

## 一、RockYou

**定位：密码字典（Password Wordlist）**

路径：

```
/usr/share/wordlists/rockyou.txt
```

压缩文件：

```
/usr/share/wordlists/rockyou.txt.gz
```

### 1. 特点

- 来源于真实泄露的密码数据
- 包含大量常见弱口令
- 主要用于密码猜测和认证测试
- 不适合直接用于 Web 目录扫描

### 2. 解压与基本操作

```
# 保留原压缩包并解压
gzip -dk /usr/share/wordlists/rockyou.txt.gz

# 查看前 20 条
head -n 20 /usr/share/wordlists/rockyou.txt

# 统计字典条目
wc -l /usr/share/wordlists/rockyou.txt

# 搜索指定内容
grep -i "admin" /usr/share/wordlists/rockyou.txt
```

### 3. 常见使用场景

| 场景         | 工具           | 参数                     |
| ------------ | -------------- | ------------------------ |
| SSH 密码测试 | Hydra          | `-P rockyou.txt`         |
| FTP 密码测试 | Hydra          | `-P rockyou.txt`         |
| SMB 密码测试 | Hydra          | `-P rockyou.txt`         |
| ZIP 密码测试 | John           | `rockyou.txt`            |
| 哈希破解     | John / Hashcat | `--wordlist=rockyou.txt` |

**Hydra 示例：**

```
hydra -l root \
-P /usr/share/wordlists/rockyou.txt \
ssh://192.168.12.136
```

**John 示例：**

```
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

## 二、SecLists

**定位：综合安全测试字典集合（Security Wordlists）**

安装：

```
apt install seclists
```

路径：

```
/usr/share/seclists/
```

### 1. 核心目录速查

| 目录           | 用途       | 典型场景             |
| -------------- | ---------- | -------------------- |
| Discovery/     | 信息发现   | Web 路径、DNS 子域名 |
| Usernames/     | 用户名字典 | 用户名枚举           |
| Passwords/     | 密码字典   | 弱口令测试           |
| Fuzzing/       | Fuzz 字典  | 参数、输入点测试     |
| Payloads/      | 攻击载荷   | 漏洞验证             |
| Miscellaneous/ | 其他字典   | 特定测试需求         |

### 2. Discovery —— 信息发现

主要用于发现目标中未知的资源。

| 子目录          | 用途                 |
| --------------- | -------------------- |
| Web-Content/    | Web 目录、文件、参数 |
| DNS/            | DNS 子域名枚举       |
| Infrastructure/ | 基础设施信息发现     |

常用字典：

```
# 常见 Web 路径
/usr/share/seclists/Discovery/Web-Content/common.txt

# 中等规模目录字典
/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt

# DNS 子域名字典
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

**Web 目录扫描：**

```
gobuster dir \
-u http://192.168.12.136 \
-w /usr/share/seclists/Discovery/Web-Content/common.txt
```

**子域名枚举：**

```
gobuster dns \
-d example.com \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

### 3. Usernames —— 用户名字典

路径：

```
/usr/share/seclists/Usernames/
```

用途：

- 常见用户名枚举
- SSH、FTP、SMB 等服务的用户测试
- Web 登录用户名测试

常用字典：

```
/usr/share/seclists/Usernames/top-usernames-shortlist.txt
```

Hydra 示例：

```
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
-P /usr/share/wordlists/rockyou.txt \
ssh://192.168.12.136
```

参数区别：

| 参数 | 含义           |
| ---- | -------------- |
| `-l` | 指定单个用户名 |
| `-L` | 指定用户名文件 |
| `-p` | 指定单个密码   |
| `-P` | 指定密码文件   |

### 4. Passwords —— 密码字典

路径：

```
/usr/share/seclists/Passwords/
```

用途：

- 常见弱口令测试
- 默认密码测试
- 特定产品或服务密码测试

常用文件：

```
/usr/share/seclists/Passwords/Common-Credentials/
```

其中包含常见凭据、弱口令及不同规模的密码列表。

与 RockYou 的区别：

| 对比 | RockYou          | SecLists Passwords   |
| ---- | ---------------- | -------------------- |
| 定位 | 经典泄露密码集合 | 多类型密码字典集合   |
| 内容 | 真实泄露密码     | 常见密码、默认凭据等 |
| 用途 | 通用密码测试     | 按场景选择密码字典   |

### 5. Fuzzing —— 模糊测试字典

路径：

```
/usr/share/seclists/Fuzzing/
```

用途：

- 参数名枚举
- 输入点测试
- 特殊字符测试
- 文件名、路径测试
- 协议及格式相关 Fuzz

例如：

```
ffuf -u "http://192.168.12.136/index.php?FUZZ=test" \
-w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt
```

`FUZZ`：表示需要被字典内容替换的位置。

### 6. Payloads —— 漏洞测试载荷

路径：

```
/usr/share/seclists/Payloads/
```

与普通字典不同，这里的内容更偏向漏洞测试输入。

常见用途：

- SQL Injection
- XSS
- LFI / Path Traversal
- 命令注入
- 文件上传测试

注意：具体目录和文件随 SecLists 版本变化，应先使用 `find` 查看。

## 三、实战场景选择

| 目标与需求     | 推荐字典              |
| -------------- | --------------------- |
| SSH 弱口令测试 | RockYou + Usernames   |
| FTP 弱口令测试 | RockYou + Usernames   |
| ZIP 密码破解   | RockYou               |
| Web 目录扫描   | Discovery/Web-Content |
| Web 后台发现   | Discovery/Web-Content |
| DNS 子域名枚举 | Discovery/DNS         |
| 用户名枚举     | Usernames             |
| Web 参数枚举   | Web-Content / Fuzzing |
| SQL 注入测试   | Payloads              |
| XSS 测试       | Payloads              |
| LFI 测试       | Payloads              |

## 四、常用字典操作命令

```
# 查看字典文件
ls -lh /usr/share/wordlists/

# 查看 SecLists 目录
ls /usr/share/seclists/

# 搜索指定字典
find /usr/share/seclists/ -iname "*common*"

# 查找所有 txt 字典
find /usr/share/seclists/ -type f -name "*.txt"

# 统计字典条目
wc -l dict.txt

# 去重
sort -u dict.txt -o dict.txt

# 查看文件大小
du -h dict.txt

# 查看文件编码和格式
file dict.txt
```

## 五、记忆锚点

| 字典      | 核心记忆 |
| --------- | -------- |
| RockYou   | 密码     |
| Discovery | 发现资源 |
| Usernames | 用户名   |
| Passwords | 密码     |
| Fuzzing   | 模糊测试 |
| Payloads  | 漏洞载荷 |

**选择原则：**

1. 先确定测试目标：密码、目录、子域名还是参数
2. 再选择对应分类的字典
3. 从小字典开始，逐步扩大范围
4. 结合目标信息定制字典，而不是盲目使用超大字典