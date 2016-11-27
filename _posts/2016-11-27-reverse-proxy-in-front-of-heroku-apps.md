---
layout: post
title: Reverse Proxy in front of Heroku Apps
comments: true
description: "前陣子試著用 reverse-proxy 把兩個架在 Heroku 上面的網站用 path 分開。本來以為應該是很單純的東西，但實際上做起來發現還有一些眉角。所以把一些遇到的問題和相關的資源記錄下來。"
tags:
  - heroku
  - "reverse-proxy"
  - haproxy
  - backend
---

前陣子試著用 [reverse-proxy](https://en.wikipedia.org/wiki/Reverse_proxy) 把兩個架在 [Heroku](https://www.heroku.com/) 上面的網站用 path 分開。
本來以為應該是很單純的東西，但實際上做起來發現還有一些眉角。所以把一些遇到的問題和相關的資源記錄下來。

## TL;DR;

- [AWS API Gateway](https://aws.amazon.com/api-gateway/) 在沒有 gzip 的狀況下可以用~~(沒有gzip誰要用阿)~~
- apigee 很貴
- Forward 到 Heroku 的時候，記得改 header 裡面的 `Host`
- 如果前端是 webapp 的話，[CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) 和 [History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API) routing 要記得改
- [references](#references)


## 情境設定

這邊想要做到的事情就是 *把兩個不同的網站用 path map 到另一個 domain*。
好比說我們的兩個網站本來是：

```
https://site-a.com
https://site-b.com
```

希望做到的結果是：

```
https://another-domain.com/site-a ==> site-a
https://another-domain.com/site-b ==> site-b
```

有一些邊界條件如下：

- site-a, site-b 都是架在 Heroku 上面
- 需要 https
- site-a 是一個 [react universal rendering](https://github.com/mz026/universal-redux-template) 的 single page app

## Solution 候選人

對於這個 reverse proxy，一開始的預期希望是可以找到一個*不用寫code的選項*，就是看有沒有 service 可以直接設定。
經過一番研究發現有可能的選項如下：

- [AWS API Gateway](https://aws.amazon.com/api-gateway/)
- [apigee](https://apigee.com/api-management/#/homepage)：看了[價格](https://apigee.com/about/pricing/apigee-edge-pricing-features)之後直接跳過

另外，還有一些要自己弄的選項：

- [HAProxy](http://www.haproxy.org/)
- [NGINX](https://www.nginx.com/resources/wiki/)

### AWS API Gateway

結論：失敗！

AWS API Gateway 在各方面看起來完全符合我們的需求，即使他的 document 真的很爛 [註1](#section-1)，
但總之我們還是就撩落去試了，結果發現

**它在處理 gzip content 的時候會出問題~~WTF!!~~**

什麼意思呢？
就是說，如果我們要 forward 的網站 （像是上面的 site-a, site-b） 回傳的是 gzipped 過的 content type,
那 **API gateway 就會搞不清楚，以為它們是 UTF8**。

這是在 [AWS forums](https://forums.aws.amazon.com/thread.jspa?threadID=212731)的回答：

> Thanks for the feedback. You are correct in that payload content is currently interpreted as UTF-8. Binary support and compression have been requested by others and are under consideration. Please stay tuned!

試了一陣子發現這種問題真的很賭藍阿！


---

接下來就是要進入自己弄的選項惹~~還是要面對阿QQ~~。

經過一番google，由於我們的需求實在是有點單純，HAProxy 和 NGINX 其實都可以做到。
但 NGINX 的功能比較多，使用起來有可能比較複雜，變數較多，
再加上我們本來的網站是架在 Heroku 上面，所以沒有已經在用哪一個，
所以就決定試試 HAProxy 惹。


## Hello HAProxy

關於 HAProxy，發現 Digital Ocean 的[教學系列](https://www.digitalocean.com/community/tutorial_series/load-balancing-wordpress-with-haproxy)相當不錯！還包括了[SSL的設定](https://www.digitalocean.com/community/tutorials/how-to-implement-ssl-termination-with-haproxy-on-ubuntu-14-04)


在照著上面的 tutorial 設定之後，發現還是不會動阿 ~~干!~~

這邊是遇到的眉角們：

## Heroku

Heroku 會用 register 在 app 上面的 [custom domain](https://devcenter.heroku.com/articles/custom-domains) 來 route，實際上是利用 request header 上面的 `Host`
所以在 HAProxy 的設定裡面，要記得改 forward 出去的 request:

```
backend site-a
  ...
  reqirep ^Host: Host:\ site-a.com
```

## Webapp

### [Histroy API](https://developer.mozilla.org/en-US/docs/Web/API/History_API) Routing

在用了新的 domain 之後，對於 webapp 來說，frontend routing 的路徑就被改變了。
好比說本來的：

```
https://site-a.com/path
```

現在變成了

```
https://another-domain.com/site-a/path
```

也就是說，對於 frontend routing 來說，同一個路徑由

```
/path
```

變成了

```
/site-a/path
```

這點要記得在 frontend 的 routing 那邊要做相對應的處理。

### CORS

如果 front-end 有做跨 domain的request [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) 的話，要記得確認新的 domain 也可以用。

### 註1

我認為 AWS 的特色就是，document 又多又難用。讓人不禁懷疑他們是不是有自己舉辦*最無用document大賽*, 如果 developer 在時間內找到需要的 document 就輸了這樣

### references:

- AWS API Gateway 哭哭：
  - [Forums](https://forums.aws.amazon.com/thread.jspa?threadID=212731)
  - [Astonishing disappointment with AWS’s API Gateway](https://medium.com/@hackupstate/astonishing-disappointment-with-awss-api-gateway-a14ea29458dc#.g3amkrn2d)

- [Digital Ocean HAProxy Tutorials](https://www.digitalocean.com/community/tutorial_series/load-balancing-wordpress-with-haproxy) (請無視 wordpress 字眼)

- ["雖然說作法很令人不解，但設定可以參考"的一篇 blog](https://pilot.co/blog/hosting-multiple-heroku-apps-on-a-single-domain/)
