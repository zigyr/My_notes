## 从部署到使用

> 拿 [API key](https://aistudio.google.com/api-keys)

```shell
npm install @google/genai
npm install -g @google/gemini-cli
$env:GEMINI_API_KEY="你的API_KEY"

$env:http_proxy="http://127.0.0.1:7890"
$env:https_proxy="http://127.0.0.1:7890"

gemini
```

> 虽然，这个 gemini 不像 agy 那般的区域限制
>
> 但是 gemini 无法走 pro 的订阅
>
> 走单独的 API key，另算非，不过辛亏，伟大的 pro 用户有 10 刀的额度
>
> 不过整体，还是不如意的
>
> 要是能用订阅，那就可以夸夸乱用了
