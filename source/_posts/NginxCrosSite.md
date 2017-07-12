title: Nginx cros site
tags:
  - nginx
categories:
  - study
cc: true
hljs: true
comments: true
date: 2017-07-12 15:39:00
---
> 又是cors跨域…cors是啥？就是跨域请求。下面给出nginx cors的相关配置。


```nginx
允许哪个域名来访问资源
add_header 'Access-Control-Allow-Origin' "$http_origin";
 
请求的返回内容里包含cookies
add_header 'Access-Control-Allow-Credentials' 'true';
 
允许请求的method
add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
```

