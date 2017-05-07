---
layout: post
title: Hello HTTPS
comments: true
description: "yay! 終於有 https 了。這篇文章要記錄的是用 Let's Encrypt 的時候遇到的狀況們。"
tags:
  - https
  - "lets-encrypt"
  - nginx
  - jekyll
---

耶！總算把 blog 加上 https 惹！看到綠色的小鎖頭就覺得爽！
在這篇文章裡面，要記錄一下安裝的過程和遇到的問題們！


## Let's Encrypt

在過去，要得到 https 的 certificate 必需要透過 Certificate Authority(CA)，像是 Godaddy 之類的。
除了要付錢給他們之外，過程基本上相當繁瑣。
Let's Encrypt 提供了一大部份的自動化的流程，讓我們可以免費的得到由它 issue 的 certificate。

但當然，這樣的過程少了真正人工的驗証（例如說確定你是不是一家合法的公司之類的），而只能在技術上確定申請的人是 domain owner。
所以在某些狀況的時候還是和付費的 CA 有差。但這篇文章不是要介紹這些差異，~~而且我也不太懂 XD~~ 所以關於 Let's Encrypt 的介紹就在此打住。

## 情境介紹

這個 blog 網站是用 [Jekyll](TODO:LINK)、[nginx](TODO:LINK) 架在 AWS 的 EC2 上面。
EC2 上面跑的是 Ubuntu。

## 安裝過程

基本上，個人認為 [Digital Ocean](TODO:LINK) 的教學非常棒，所以只要跟著上面的步驟大致上是沒什麼問題的。
那這篇文章豈不是廢文一篇了嗎？

## nginx 的版本問題

在 Digital Ocean 的教學裡面，有使用到一些設定是較新版本的 nginx 才有的，像是 http2 的設定。
所以如果設定跑不起來，可以檢查看看是不是 nginx 版本的問題。

## 要怎麼處理不認識的 domain？

當我們把 blog 架起來之後，要怎麼"不讓別人從別的 domain 來 access" 呢？
以我的狀況為例，我希望只有從 `blog.mz026.rocks` 才可以連到這個 blog。

但如果有人買了另一個 domain `blog.mz027-evil.rocks`，然後把這個 domain 也指到我的機器的 IP,
那我要怎麼不讓這個邪惡的 domain 綁架我的內容呢？

在加上 https 之前，我是用這樣的設定來做到：

```
# /etc/nginx/sites-enabled/blog.conf

server {
  listen 80;
  return 403;
}

server {
  listen 80;
  server_name blog.mz026.rocks;

  # ... other settings
}
```

這樣一來，從其他 domain 企圖連進來的都會得到 `403 (forbidden)` 的結果。

但加上 https 之後，如果


