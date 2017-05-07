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

這個 blog 網站是用 [Jekyll](https://jekyllrb.com/)、[nginx](https://nginx.org/en/) 架在 AWS 的 EC2 上面。
EC2 上面跑的是 Ubuntu。

## 安裝過程

基本上，個人認為 [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04) 的教學非常棒，所以只要跟著上面的步驟大致上是沒什麼問題的。
~~hmm...那這篇文章豈不是廢文一篇了嗎？~~

但過程中還是有遇到一些小小問題，整理如下：

### nginx 的版本問題

在 Digital Ocean 的教學裡面，有使用到一些設定是較新版本的 nginx 才有的，像是 http2 的設定。
所以如果設定跑不起來，可以檢查看看是不是 nginx 版本的問題。

### 要怎麼處理不認識的 domain？

當我們把 blog 架起來之後，要怎麼"不讓別人從別的 domain 來 access" 呢？
以我的狀況為例，我希望只有從 `blog.mz026.rocks` 才可以連到這個 blog。

但如果有人買了另一個 domain `blog.mz027-evil.rocks`，然後把這個 domain 也指到我的機器的 IP,
那我要怎麼不讓這個邪惡的 domain 綁架我的內容呢？

在加上 https 之前，我是用這樣的設定來做到：

```conf
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

但加上 https 之後，如果用 `https://<ip.address>` 連進來的話，卻不會得到預期的 403，而是會得到 certificate 不符合之類的東西。

{: .center}
![ssl-444](/images/hello-https/ssl-not-private.png)

但是如果 user 想要的話，還是可以不顧一切進去看到內容。
但我希望的是如果是不認識的 domain 就什麼都不給看。

在經過一番 google 之後，後來是在 nginx 的 config 裡面判斷 host name 來決定：

```conf
server {
        listen 443 ssl;
        server_name blog.mz026.rocks;

        # ... other config

        if ( $http_host != $server_name ) {
            return 444;
        }
}
```

其本上就是，如果來的 host 不是自己的 domain 的話，就回一個不合法的 http status code。
(其中的 `444` 是來自於[這邊](http://nginx.org/en/docs/http/request_processing.html))

這樣的話前端如果不管 warning 硬要進去，就會看到這種東西：

{: .center}
![ssl-444](/images/hello-https/ssl-444.png)

這樣算是有達到我預期的成果，就是不認識的 domain 不要亂來這樣。

### Addthis 的 facebook like 數字歸零了 QQ

在每篇文章的最下面，我有放上 [Addthis](TODO:LINK) 作為社群分享的外掛。
但是在換到 https 之後，上面本來的數據都歸零了 QQ
原因是因為本來的數據是記在 `http://...` 這些網址上的，
但現在這些 `http` 開頭的網址，都被我用 `301` 導到 `https` 的版本，
而在 `https` 版本的文章，就會試圖去找 `https://...` 在 facebook 上面的數據，於是就什麼都沒有拿到了。

後來解決的方法是：

**1: 修改 Jekyll 的 template**

在 Jekyll 的 template 上面，去判斷文章的時間。如果早於"換成 https"之前的文章，就把 `og:url` 的 `<meta>` 設成 `http` 版本的 url。
反之則用 `https` 版本:

下面是 `_head.html` 的部份內容：

```
{% capture page_time %}{{page.date | date: '%y%m%d'}}{% endcapture %}
{% assign ssl_transfer_time = '170506' %}

{% if page_time < ssl_transfer_time %}
  <meta property="og:url" content="{{ site.http_url }}{{ page.url }}">
{% else %}
  <meta property="og:url" content="{{ site.url }}{{ page.url }}">
{% endif %}
```

**2: 修改 Addthis 的 javascript 設定**

在 Addthis 的 javascript 設定裡面，去找 `og:url` 的網址當成 url。
以我的設定為例，下面是 `_post.html` 的部份內容：

```html
<script type="text/javascript" charset="utf-8">
  var addthis_share = {
    url: (function() {
      return document.querySelectorAll("meta[property='og:url']")[0].content
    }()),
    title: '{{ page.title }}'
  }
</script>
```


## 心得與感想

- Digital Ocean 有好多這類的教學都很棒！
- nginx 的設定我實在是很不會，但感覺這個東西真的是博大精深。
- Jekyll 的 template 用的是 [Liquid template](https://shopify.github.io/liquid/) (shopify 做的)，個人覺得有夠難用。但好險要修改的機會也不高就是了，不然真的是會抓狂。
