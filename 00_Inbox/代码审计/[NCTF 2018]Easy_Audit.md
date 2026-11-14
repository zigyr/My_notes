```php
<?php
highlight_file(__FILE__);
error_reporting(0);
if($_REQUEST){
    foreach ($_REQUEST as $key => $value) {
        if(preg_match('/[a-zA-Z]/i', $value))   die('waf..');
    }
}

if($_SERVER){
    if(preg_match('/yulige|flag|nctf/i', $_SERVER['QUERY_STRING']))  die('waf..');
}

if(isset($_GET['yulige'])){
    if(!(substr($_GET['yulige'], 32) === md5($_GET['yulige']))){         //日爆md5!!!!!!
        die('waf..');
    }else{
        if(preg_match('/nctfisfun$/', $_GET['nctf']) && $_GET['nctf'] !== 'nctfisfun'){
            $getflag = file_get_contents($_GET['flag']);
        }
        if(isset($getflag) && $getflag === 'ccc_liubi'){
            include 'flag.php';
            echo $flag;
        }else die('waf..');
    }
}
?>
```



`$_REQUEST` 是 PHP 的一个超全局变量，主要作用是将 GET、POST、COOKIE 中的请求参数合并到一个数组中

它最大的特点是：同名参数发生冲突时，最终保留哪个值，取决于 PHP 配置

> PHP 主要通过 `request_order` 配置控制 `$_REQUEST` 的合并顺序
>
> | 字母 | 含义   |
> | ---- | ------ |
> | G    | GET    |
> | P    | POST   |
> | C    | COOKIE |
>
> 常见默认值：`request_order = "GP"`，先合并 GET，再合并 POST
>
> **后面出现的同名参数会覆盖前面的值**

而这里通过 `$_REQUEST` 获取 `$key`、`$value`

并对，`$value` 进行无字母检测

此时就可以通过在 POST 传参来瞒天过海





`$_SERVER['QUERY_STRING']` 用来获取 URL 中 `?` 后面的原始查询字符串

重点：它不会像 `$_GET` 一样自动进行 URL 解码

```php
<?php

var_dump($_SERVER['QUERY_STRING']);
var_dump($_GET);

/**
string(18) "%6E%61%6D%65=zigyr"

array(1) {
  ["name"]=> string(5) "zigyr"
}
```

所以，这里我们 GET 传参的数据就可以通过 URL 编码进行绕过





`if(!(substr($_GET['yulige'], 32) === md5($_GET['yulige'])))`

输入第 33 个字符之后的内容，必须严格等于整个输入的 MD5 值

payload：`?yulige[]=1`

> 在 PHP 7 等旧版本中：
>
> 1. `substr()` 接收到数组，产生警告并返回 `NULL`
> 2. `md5()` 接收到数组，同样产生警告并返回 `NULL`





`if(preg_match('/nctfisfun$/', $_GET['nctf']) && $_GET['nctf'] !== 'nctfisfun')`

既要求 preg_match 匹配到 nctfisfun，有要求不能等于 nctfisfun

利用换行符绕过正则，当传入 `nctfisfun\n`

由于正则表达式中的 `$` 在默认模式下，匹配字符串末尾的最后一个换行符之前的位置

所以，`preg_match('/nctfisfun$/', "nctfisfun\n")` 为 true

而 `"nctfisfun\n" !== "nctfisfun"` 因为多了一个换行符，两个字符串并不完全相同，也为 true

故，payload：`?nctfisfun=nctfisfun\n`



`$getflag = file_get_contents($_GET['flag']);`

要让 `$getflag` 等于 ccc_liubi

可以利用 data 伪协议

payload：`?flag=data://text/plain,ccc_liubi`



综上，结合 `$REQUEST`、`$_SERVER['QUERY_STRING']`

最终 payload：

```
GET :?%79%75%6C%69%67%65[]=1&%6E%63%74%66=%6E%63%74%66%69%73%66%75%6E%0a&%66%6C%61%67=data://text/plain,ccc_liubi

POST: yulige=1&nctf=2&flag=1
```

得到 flag![image-20261112102231662](%5BNCTF%202018%5DEasy_Audit/image-20261112102231662.png)
