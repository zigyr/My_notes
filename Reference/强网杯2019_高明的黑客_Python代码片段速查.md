# Python 自动化脚本常用代码片段

## 总结

这份 EXP 涉及三类常见编程任务：

1. **数据处理**：文件遍历、分组、正则提取、有序去重
2. **网络请求**：GET / POST 参数批量构造、多阶段检测
3. **任务调度**：多线程分发、Event 状态控制

这些代码片段可以组合用于 CTF Web 自动化探测、批量源码分析和网络请求检测

## 1. 常用语法速查

| 语法                       | 作用             |
| -------------------------- | ---------------- |
| `Path.iterdir()`           | 遍历目录         |
| `Path.name`                | 获取文件名       |
| `Path.suffix`              | 获取文件后缀     |
| `str.rstrip('/')`          | 去除右侧指定字符 |
| `enumerate(items)`         | 遍历并获取索引   |
| `zip(a, b)`                | 并行遍历多个序列 |
| `range(start, stop, step)` | 按步长生成序列   |
| `list[start:end]`          | 列表切片         |

## 2. 文件批量分组

**适用场景：** 将大量文件按照指定数量分组，分配给不同线程处理

```python
from pathlib import Path

def get_file(path, group_size):
    files = list(path.iterdir())
    return [
        files[i:i + group_size]
        for i in range(0, len(files), group_size)
    ]
```

## 3. 正则提取与有序去重

**适用场景：** 从 PHP 源码、日志等文本中批量提取指定内容

```python
import re

results = re.findall(pattern, content)
results = list(dict.fromkeys(results))
```

- `re.findall()`：提取所有匹配项
- `dict.fromkeys()`：去重并保留原始顺序

## 4. GET / POST 参数批量构造

**适用场景：** 将提取出的参数名批量转换为 HTTP 请求参数

```python
PAYLOAD = "test"

params = {key: PAYLOAD for key in gets}
data = {key: PAYLOAD for key in posts}
```

- `params`：作为 GET 查询参数
- `data`：作为 POST 请求体参数

## 5. 多阶段检测

**适用场景：** 大量候选目标的自动化筛选与精准定位

- 第一阶段：批量提交候选参数，快速筛选
- 第二阶段：逐个测试，定位具体命中项

核心思路：粗筛（Bulk Scan）→ 精筛（Fine-grained Verification）

## 6. 多线程任务分发

**适用场景：** 并发执行网络请求、文件处理等 I/O 密集型任务

```python
threads = []

for task in tasks:
    t = threading.Thread(target=worker, args=(task,))
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

- `start()`：启动线程
- `join()`：等待线程结束
- `args=(task,)`：传递单个参数，注意逗号不能省略

## 7. Event 多线程状态控制

**适用场景：** 某个线程发现目标后，通知其他线程停止后续任务

```python
STOP_EVENT = threading.Event()

# 设置停止标志
STOP_EVENT.set()

# 检查停止状态
if STOP_EVENT.is_set():
    return

# 重置状态
STOP_EVENT.clear()
```