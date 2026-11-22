

## 从安装到使用：

```powershell
irm https://antigravity.google/cli/install.ps1 | iex

$env:Path += ";$env:LOCALAPPDATA\agy\bin"
agy --version

$env:HTTP_PROXY="http://192.168.163.78:7890"
$env:HTTPS_PROXY="http://192.168.163.78:7890"

agy
```

## 关于 agy 的认证：

更改地区

- 查看账号归属地

https://policies.google.com/terms

- 申请修改账号地区

https://policies.google.com/country-association-form

- 填写申请表单

  - 选择 "**Other reason**"
  - 写明需要访问美国独家开发工具的理由，并请求修改地区

  ```text
  I have relocated to Los Angeles, California, USA, and would like to update the country/region associated with my Google Account to reflect my current location.
  
  I need access to Google developer tools and related services for my software development work. Please assist me with this update.
  
  Thank you for your help.
  ```


> 但是，可恨，因为代理出口很乱的原因
>
> 尝试了两次，都未能成功