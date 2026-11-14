# Linux Shell 文本处理速查

## 1. 常见处理套路

### 从 Nmap 结果中提取开放端口

``` bash
grep -E '^[0-9]+/tcp[[:space:]]+open' nmap.txt
```

筛选 TCP 开放端口所在行

``` bash
grep -E '^[0-9]+/tcp[[:space:]]+open' nmap.txt \
  | awk -F '/' '{print $1}' \
  | sort -nu \
  | paste -sd ',' -
```

筛选开放端口 → 提取端口号 → 数值排序并去重 → 合并成逗号分隔的一行

### 从日志中查找错误

``` bash
grep -inE 'error|failed|denied' app.log
```

忽略大小写，显示行号，匹配多个关键词

### 提取 CSV 某一列

``` bash
awk -F ',' '{print $2}' data.csv
```

以逗号分隔，输出第二列。若 CSV 含带引号的逗号字段，应使用专门的 CSV
工具，不能简单依赖 `awk -F ','`

### 去重并排序

``` bash
sort -u users.txt       # 字典序排序并去重
sort -nu ports.txt      # 数值排序并去重
sort -rn scores.txt     # 数值逆序排序
```

### 多行合并为一行

``` bash
paste -sd ',' ports.txt
```

把多行内容合并为逗号分隔的一行

------------------------------------------------------------------------

## 2. grep：按内容筛选行

**用途：** 从文件或管道输入中找出匹配的行

基本语法：

``` bash
grep [选项] '匹配内容' 文件
```

常用参数：

| 参数 | 作用 |
|---|---|
| `-i` | 忽略大小写 |
| `-v` | 反向匹配，排除匹配行 |
| `-n` | 显示行号 |
| `-E` | 使用扩展正则表达式 |
| `-o` | 只输出匹配到的部分 |
| `-c` | 统计匹配行数 |
| `-r` | 递归搜索目录 |

常用 Demo：

``` bash
grep 'root' /etc/passwd          # 查找包含 root 的行
grep -in 'error' app.log         # 忽略大小写并显示行号
grep -v 'debug' app.log          # 排除包含 debug 的行
grep -E 'error|failed' app.log   # 匹配 error 或 failed
grep -oE '[0-9]+' file.txt       # 只输出匹配到的数字片段
grep -c '成功' result.txt        # 统计匹配行数
grep -rni 'password' ./          # 在目录中递归搜索
```

常见正则片段：

| 表达式 | 含义 |
|---|---|
| `.` | 任意单个字符 |
| `*` | 前一个字符重复 0 次或多次 |
| `^` | 行首 |
| `$` | 行尾 |
| `[0-9]` | 一个数字 |
| `[[:space:]]` | 空白字符 |
| `a|b`（基本正则） | `a` 或 `b` |
| `a|b`（扩展正则 `-E`） | `a` 或 `b` |

使用 `grep -E` 时，通常直接写 `a|b`：

``` bash
grep -E 'ssh|http' services.txt
```

------------------------------------------------------------------------

## 3. awk：按字段处理文本

**用途：** 按分隔符拆分每一行，提取字段、按条件筛选、统计和格式化输出

基本语法：

``` bash
awk -F '分隔符' '条件 {动作}' 文件
```

### 3.1 字段与内置变量

| 写法 | 含义 |
|---|---|
| `$0` | 当前整行 |
| `$1`、`$2` | 第 1、2 个字段 |
| `$NF` | 当前行最后一个字段 |
| `NF` | 当前行字段数量 |
| `NR` | 当前处理的总行号 |
| `FNR` | 当前文件中的行号 |
| `-F` | 指定输入字段分隔符 |
| `OFS` | 输出字段分隔符 |
| `ORS` | 输出记录分隔符 |

### 3.2 提取字段

``` bash
awk '{print $1}' file.txt             # 默认按空白分隔，输出第一列
awk '{print $1, $3}' file.txt         # 输出第一列和第三列
awk '{print $NF}' file.txt            # 输出最后一列
awk -F '/' '{print $1}' ports.txt     # 按 / 分隔，提取端口号
awk -F ':' '{print $1}' /etc/passwd   # 按 : 分隔，提取第一列
```

### 3.3 按条件筛选

``` bash
awk '$2 == "open" {print $1}' nmap.txt # 第二列等于 `open` 时，输出第一列
awk '$3 > 100 {print $1, $3}' data.txt # 第三列大于 100 时，输出第一列和第三列
awk 'NR > 1 {print $1}' data.txt       # 跳过第一行，从第二行开始处理
```

### 3.4 格式化输出

``` bash
awk '{print "Port:", $1}' ports.txt
awk 'BEGIN {OFS=":"} {print $1, $2}' file.txt
```

`BEGIN` 在正式处理输入前执行；`OFS` 设置输出字段之间的分隔符

### 3.5 统计与汇总

``` bash
awk 'END {print NR}' file.txt           # 统计总行数
awk '{count++} END {print count}' file.txt
awk '{sum += $1} END {print sum}' nums.txt  # 对第一列求和
```

### 3.6 常见结构

``` bash
awk 'BEGIN {print "Start"} {print $1} END {print "Done"}' file.txt
```

-   `BEGIN`：读取文件前执行一次
-   主体 `{...}`：对每一行执行
-   `END`：所有行处理完后执行一次
-   `print`：输出内容，是 awk 的语句，不是独立 Shell 命令

------------------------------------------------------------------------

## 4. sort：排序与去重

**用途：** 对文本行排序，常与 `uniq` 或 `awk` 配合

| 参数 | 作用 |
|---|---|
| `-n` | 按数值排序 |
| `-r` | 逆序 |
| `-u` | 排序并去重 |
| `-k` | 指定排序字段 |
| `-t` | 指定字段分隔符 |
| `-f` | 忽略大小写 |

常用 Demo：

``` bash
sort names.txt                 # 默认字典序
sort -n ports.txt              # 数值升序
sort -rn scores.txt            # 数值降序
sort -u users.txt              # 排序并去重
sort -t ':' -k 3,3n passwd.txt # 按 : 分隔，按第 3 列数值排序
```

> 注意：端口号、数值等通常使用 `-n`，否则可能按字符顺序排列

------------------------------------------------------------------------

## 5. uniq：相邻重复行处理

**用途：** 删除或统计连续重复的行。通常先 `sort`，让相同内容相邻

``` bash
sort users.txt | uniq          # 去掉相邻重复行
sort users.txt | uniq -c       # 统计每种内容出现次数
sort users.txt | uniq -d       # 只显示重复内容
sort users.txt | uniq -u       # 只显示未重复内容
```

------------------------------------------------------------------------

## 6. tr：字符转换、删除与压缩

**用途：** 处理字符，不负责理解字段结构

| 参数 | 作用 |
|---|---|
| `-d` | 删除指定字符 |
| `-s` | 压缩连续重复字符 |
| `-c` | 对指定字符集取补集 |

常用 Demo：

``` bash
echo 'HELLO' | tr 'A-Z' 'a-z'      # 大写转小写
echo 'hello' | tr 'a-z' 'A-Z'      # 小写转大写
echo 'a    b' | tr -s ' '         # 连续空格压缩成一个
echo '22/tcp' | tr -d '/tcp'       # 删除字符集中的字符（不是删除子串）
tr -d '\r' < windows.txt           # 删除回车符
```

换行转逗号：

``` bash
tr '\n' ',' < ports.txt
```

如果希望合并多行并避免最后多一个逗号，通常优先使用：

``` bash
paste -sd ',' ports.txt
```

------------------------------------------------------------------------

## 7. cut：按分隔符或字符位置截取

**用途：** 简单、固定格式文本的列提取

| 参数 | 作用 |
|---|---|
| `-d` | 指定分隔符 |
| `-f` | 指定字段 |
| `-c` | 指定字符位置 |

``` bash
cut -d ':' -f 1 /etc/passwd   # 提取冒号分隔的第一列
cut -d '/' -f 1 ports.txt     # 提取端口号
cut -d ',' -f 1,3 data.csv    # 提取第 1、3 列
cut -c 1-5 file.txt           # 提取每行第 1 到 5 个字符
```

------------------------------------------------------------------------

## 8. sed：按规则替换或处理行

**用途：** 常见于文本替换、删除行、只打印指定行

``` bash
sed 's/old/new/' file.txt          # 每行替换第一个 old
sed 's/old/new/g' file.txt         # 每行替换所有 old
sed -n '5p' file.txt               # 只打印第 5 行
sed -n '2,8p' file.txt             # 打印第 2 到 8 行
sed '/debug/d' file.txt            # 删除包含 debug 的行（输出中删除）
sed -i 's/old/new/g' file.txt      # 直接修改文件，使用前先备份
```

默认情况下 `sed` 将处理结果输出到标准输出；`-i` 会直接修改文件

------------------------------------------------------------------------

## 9. paste：按行合并内容

**用途：** 将多个文件或多行内容按列合并

``` bash
paste file1.txt file2.txt          # 两个文件按行合并，默认用 Tab 分隔
paste -d ',' file1.txt file2.txt   # 使用逗号分隔
paste -sd ',' ports.txt            # 将单个文件的多行合并成一行
```

------------------------------------------------------------------------

## 10. xargs：把标准输入转成命令参数

**用途：** 将管道输入交给另一个命令作为参数

``` bash
cat urls.txt | xargs -n 1 ping -c 1
```

每次取一个输入项，作为 `ping` 的参数

``` bash
printf '%s\n' file1 file2 | xargs -n 1 ls -l
```

逐个对输入项执行 `ls -l`

文件名可能包含空格或特殊字符时，优先使用 `find -print0` 配合
`xargs -0`：

``` bash
find . -type f -name '*.log' -print0 | xargs -0 grep -n 'error'
```

------------------------------------------------------------------------

## 11. 管道、重定向与命令替换

### 11.1 管道 `|`

把前一个命令的标准输出传给下一个命令的标准输入

``` bash
grep 'open' nmap.txt | awk '{print $1}
```

典型处理链：

``` bash
cat file.txt | grep '关键词' | sort -u
```

如果命令本身支持直接读取文件，通常不需要额外使用 `cat`：

``` bash
grep '关键词' file.txt | sort -u
```

### 11.2 重定向

``` bash
command > out.txt       # 覆盖写入标准输出
command >> out.txt      # 追加标准输出
command 2> error.txt    # 标准错误写入文件
command > all.txt 2>&1  # 标准输出和标准错误写入同一文件
```

### 11.3 命令替换 `$()`

执行括号内命令，并将其标准输出替换到当前位置

``` bash
PORTS=$(printf '22,80,443')
echo "$PORTS"
```

推荐使用 `$()`，比反引号更易读，也更方便嵌套

### 11.4 变量

``` bash
IP=192.168.56.115    # 定义变量，等号两侧不能有空格
echo "$IP"           # 引用变量
nmap -sV "$IP"       # 在命令中使用变量
```

建议用双引号包裹变量，避免空格或通配符导致意外拆分

------------------------------------------------------------------------

## 12. 综合 Demo：处理 Nmap 扫描结果

假设 `nmap.txt` 中有如下内容：

``` text
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
443/tcp  closed https
3306/tcp open  mysql
```

### 提取开放 TCP 端口

``` bash
grep -E '^[0-9]+/tcp[[:space:]]+open' nmap.txt \
  | awk -F '/' '{print $1}' \
  | sort -nu \
  | paste -sd ',' -
```

输出：

``` text
22,80,3306
```

### 统计开放端口数量

``` bash
grep -Ec '^[0-9]+/tcp[[:space:]]+open' nmap.txt
```

### 提取服务名称

``` bash
awk '$2 == "open" {print $3}' nmap.txt
```

### 保存端口列表到变量

``` bash
PORTS=$(grep -E '^[0-9]+/tcp[[:space:]]+open' nmap.txt \
  | awk -F '/' '{print $1}' \
  | sort -nu \
  | paste -sd ',' -)

printf '%s\n' "$PORTS"
```

------------------------------------------------------------------------

## 13. 函数速记

| 命令 / 结构 | 核心用途 |
|---|---|
| `grep` | 按内容筛选行 |
| `awk` | 按字段提取、条件判断、统计、格式化 |
| `sort` | 排序、去重 |
| `uniq` | 处理相邻重复行、计数 |
| `tr` | 字符转换、删除、压缩 |
| `cut` | 按分隔符或字符位置截取 |
| `sed` | 文本替换、删除或选取行 |
| `paste` | 按行合并或将多行合成一行 |
| `xargs` | 将标准输入转换为命令参数 |
| `print` | awk 内部输出语句 |
| `\|` | 管道，连接命令输入输出 |
| `>` / `>>` | 覆盖 / 追加重定向 |
| `$()` | 命令替换 |
| `$变量` | 引用 Shell 变量 |

**记忆思路：** `grep` 找行，`awk` 拆字段，`cut` 截列，`sort/uniq`
排序去重，`sed` 替换行，`tr` 改字符，`paste` 合并行，`xargs` 传参数
