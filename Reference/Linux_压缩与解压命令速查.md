# **Linux 压缩与解压命令速查**

## **核心结论**

Linux 中的压缩与解压，主要记住三类工具：

| 工具类型   | 常用命令                      | 核心作用                           |
| ---------- | ----------------------------- | ---------------------------------- |
| 单文件压缩 | `gzip`、`bzip2`、`xz`、`zstd` | 压缩单个文件                       |
| 打包与压缩 | `tar`                         | 将多个文件、目录打包，并可同时压缩 |
| 压缩包管理 | `zip`、`unzip`、`7z`          | 直接压缩、解压文件和目录           |

**最值得记住的 5 条命令：**

```shell
tar -czf backup.tar.gz dir/    # 打包并使用 gzip 压缩
tar -xzf backup.tar.gz         # 解压 tar.gz
unzip file.zip                 # 解压 ZIP
7z x file.7z                   # 解压 7z
file filename                  # 识别文件真实格式
```

核心区别：

- `.gz`：单文件压缩
- `.tar`：文件打包，**不压缩**
- `.tar.gz`：先使用 tar 打包，**再使用 gzip 压缩**
- `.zip`、`.7z`：可以直接将多个文件和目录加入压缩包

## **1. 常见格式命令速查**

| 格式       | 解压命令                      | 压缩命令                           |
| ---------- | ----------------------------- | ---------------------------------- |
| `.gz`      | `gzip -d file.gz`             | `gzip file`                        |
| `.bz2`     | `bzip2 -d file.bz2`           | `bzip2 file`                       |
| `.xz`      | `xz -d file.xz`               | `xz file`                          |
| `.zst`     | `zstd -d file.zst`            | `zstd file`                        |
| `.zip`     | `unzip file.zip`              | `zip -r file.zip dir/`             |
| `.7z`      | `7z x file.7z`                | `7z a file.7z dir/`                |
| `.tar`     | `tar -xf file.tar`            | `tar -cf file.tar dir/`            |
| `.tar.gz`  | `tar -xzf file.tar.gz`        | `tar -czf file.tar.gz dir/`        |
| `.tar.bz2` | `tar -xjf file.tar.bz2`       | `tar -cjf file.tar.bz2 dir/`       |
| `.tar.xz`  | `tar -xJf file.tar.xz`        | `tar -cJf file.tar.xz dir/`        |
| `.tar.zst` | `tar --zstd -xf file.tar.zst` | `tar --zstd -cf file.tar.zst dir/` |

> **注意：**
>
> - `gzip`、`bzip2`、`xz`、`zstd` 通常处理单个文件，不负责将整个目录打包
> - `tar` 负责打包；加上 `-z`、`-j`、`-J`、`--zstd` 才会使用对应压缩算法
> - 普通压缩命令通常会删除原文件，使用 `-k` 可以保留原文件（具体支持情况以工具为准）

## **2. 按实际场景使用**

### **2.1 解压到当前目录**

```shell
gzip -d file.gz
bzip2 -d file.bz2
xz -d file.xz
zstd -d file.zst

unzip file.zip
7z x file.7z

tar -xf file.tar
tar -xzf file.tar.gz
tar -xjf file.tar.bz2
tar -xJf file.tar.xz
tar --zstd -xf file.tar.zst
```

### **2.2 解压到指定目录**

```shell
mkdir -p ./output

unzip file.zip -d ./output
7z x file.7z -o./output

tar -xf file.tar -C ./output
tar -xzf file.tar.gz -C ./output
```

### **2.3 压缩单个文件**

```shell
gzip file.txt   -> file.txt.gz
bzip2 file.txt  -> file.txt.bz2
xz file.txt     -> file.txt.xz
zstd file.txt   -> file.txt.zst
```

保留原文件：

```shell
gzip -k file.txt
bzip2 -k file.txt
xz -k file.txt
zstd -k file.txt
```

### **2.4 压缩整个目录**

```shell
tar -cf backup.tar dir/              # 只打包
tar -czf backup.tar.gz dir/          # tar + gzip
tar -cjf backup.tar.bz2 dir/         # tar + bzip2
tar -cJf backup.tar.xz dir/          # tar + xz
tar --zstd -cf backup.tar.zst dir/   # tar + zstd

zip -r backup.zip dir/               # ZIP 递归压缩
7z a backup.7z dir/                  # 7z 添加目录
```

### **2.5 查看压缩包内容，不实际解压**

```shell
tar -tf file.tar
tar -tzf file.tar.gz
tar -tjf file.tar.bz2
tar -tJf file.tar.xz
tar --zstd -tf file.tar.zst

unzip -l file.zip
7z l file.7z
zipinfo file.zip
```

### **2.6 指定压缩级别**

```shell
gzip -1 file.txt     # 较快，压缩率较低
gzip -9 file.txt     # 较慢，通常压缩率较高

xz -1 file.txt
xz -9 file.txt

zstd -1 file.txt
zstd -19 file.txt
```

> 压缩级别越高，通常耗时越长、压缩率越高
>
> 实际效果取决于文件内容

## **3. 常用参数速查**

### **3.1 gzip / bzip2 / xz / zstd**

| 参数       | 含义                               |
| ---------- | ---------------------------------- |
| `-d`       | 解压（Decompress）                 |
| `-k`       | 保留原文件（Keep）                 |
| `-c`       | 输出到标准输出，不直接替换输入文件 |
| `-t`       | 测试压缩文件完整性                 |
| `-1`～`-9` | 常见压缩级别，具体范围依工具而定   |
| `-v`       | 显示处理信息（Verbose）            |

### **3.2 tar**

| 参数     | 含义                   |
| -------- | ---------------------- |
| `-c`     | 创建归档（Create）     |
| `-x`     | 解开归档（Extract）    |
| `-t`     | 列出归档内容（List）   |
| `-f`     | 指定归档文件名（File） |
| `-z`     | 使用 gzip              |
| `-j`     | 使用 bzip2             |
| `-J`     | 使用 xz                |
| `-v`     | 显示处理过程           |
| `-C`     | 指定操作目录           |
| `--zstd` | 使用 zstd              |

常见组合：

```shell
tar -czvf backup.tar.gz dir/       # 创建并显示过程
tar -xzvf backup.tar.gz            # 解压并显示过程
tar -tzvf backup.tar.gz            # 查看归档内容
tar -xzf backup.tar.gz -C ./out    # 解压到指定目录
```

记忆：

- `c`：创建
- `x`：解压
- `t`：查看
- `z`：gzip
- `j`：bzip2
- `J`：xz
- `f`：指定文件名

### **3.3 zip / unzip**

| 工具    | 参数 | 含义                         |
| ------- | ---- | ---------------------------- |
| `zip`   | `-r` | 递归压缩目录                 |
| `zip`   | `-e` | 创建加密 ZIP，交互式输入密码 |
| `zip`   | `-q` | 静默模式                     |
| `unzip` | `-l` | 查看压缩包内容               |
| `unzip` | `-d` | 指定解压目录                 |
| `unzip` | `-o` | 覆盖已有文件，不询问         |
| `unzip` | `-n` | 不覆盖已有文件               |

### **3.4 7z**

| 命令 / 参数 | 含义                          |
| ----------- | ----------------------------- |
| `a`         | 添加文件或创建压缩包          |
| `x`         | 解压并保留目录结构            |
| `e`         | 解压但不保留目录结构          |
| `l`         | 列出压缩包内容                |
| `-oDIR`     | 指定输出目录，目录名紧跟 `-o` |
| `-p`        | 设置密码                      |

示例：

```shell
7z x file.7z -o./output
7z a archive.7z dir/
7z l archive.7z
```

## **4. 文件识别、检查与辅助命令**

```shell
file filename               # 根据文件内容识别类型
ls -lh                      # 查看文件大小
du -sh dir/                 # 查看目录占用空间
find . -type f -name '*.gz' # 查找当前目录下的 gz 文件
```

完整性检查：

```shell
gzip -t file.gz
bzip2 -t file.bz2
xz -t file.xz
zstd -t file.zst

unzip -t file.zip
7z t file.7z
```

`file` 会根据文件内容特征判断类型，而不只是依赖扩展名。

遇到未知文件时，优先执行：

```shell
file unknown_file
```

## **5. 容易混淆的地方**

### **5.1** `.gz` **和** `.tar.gz` **不一样**

```
gzip file.txt
```

得到 `file.txt.gz`，这是单个文件的 gzip 压缩结果

```shell
tar -czf archive.tar.gz dir/
```

得到 `archive.tar.gz`，其中包含目录结构和文件，再经过 gzip 压缩

### **5.2** `tar -f` **后面跟归档文件名**

```shell
tar -czf archive.tar.gz dir/
```

- `-c`：创建归档
- `-z`：使用 gzip
- `-f archive.tar.gz`：指定输出归档文件
- `dir/`：要打包的目录

### **5.3** `7z x` **和** `7z e` **的区别**

```shell
7z x archive.7z  # 保留压缩包内的目录结构
7z e archive.7z  # 不保留目录结构，文件解到同一层
```

### **5.4 覆盖文件要谨慎**

```shell
unzip -o file.zip
```

直接覆盖同名文件，不询问。

不希望覆盖已有文件：

```shell
unzip -n file.zip
```

## **6. 一页记忆重点**

| 需求           | 命令                           |
| -------------- | ------------------------------ |
| 解压 `.gz`     | `gzip -d file.gz`              |
| 解压 `.zip`    | `unzip file.zip`               |
| 解压 `.7z`     | `7z x file.7z`                 |
| 解压 `.tar.gz` | `tar -xzf file.tar.gz`         |
| 解压 `.tar.xz` | `tar -xJf file.tar.xz`         |
| 创建 `.zip`    | `zip -r file.zip dir/`         |
| 创建 `.7z`     | `7z a file.7z dir/`            |
| 创建 `.tar.gz` | `tar -czf file.tar.gz dir/`    |
| 查看归档内容   | `tar -tf file.tar`             |
| 解压到指定目录 | `tar -xf file.tar -C ./output` |
| 识别文件类型   | `file filename`                |
| 测试 gzip 文件 | `gzip -t file.gz`              |

**参数记忆：**

- `-d`：Decompress，解压
- `-k`：Keep，保留原文件
- `-r`：Recursive，递归处理目录
- `-v`：Verbose，显示详细过程
- `-f`：File，在 `tar` 中用于指定归档文件
- `-C`：指定操作目录，常用于 `tar` 解压
- `-p`：在 `7z` 中用于密码相关设置，不同工具中含义可能不同

**最终记忆：**

单文件压缩用 `gzip / bzip2 / xz / zstd`；目录打包并压缩用 `tar`；跨平台分享常用 `zip`；需要多种压缩格式支持时可用 `7z`
