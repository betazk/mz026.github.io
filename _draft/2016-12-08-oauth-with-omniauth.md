---
layout: post
title: Integrate OAuth with Omniauth
comments: true
description: "在這篇文章裡，想跟大家分享的是 OAuth 的一些基本介紹，以及使用 omniauth 這個 ruby gem 去接 3rd-party 的 auth API"
tags:
  - oauth
  - omniauth
  - backend
  - rails
---

在很多時候，我們在做 Login/Signup 的功能的同時，會希望也一併加入 Social Connect 的功能。
好比說除了最基本的 email/password 登入以外，我們也會想要加入 Facebook 或 Twitter login。
大部份像是 Facebook 或是 Twitter 這類的 ~~大!~~ 平台，都有提供 [OAuth](TODO:LINK) 的 API。

在這篇文章裡，想跟大家分享的是：

- OAuth 的一些基本介紹
- 使用 [omniauth](TODO:LINK) 這個 ruby gem 去接 3rd-party 的 auth API，包括：
  - [omniauth](TODO:LINK) 在整個 server app 裡扮演的角色
  - omniauth 內部的結構和運作方式
  - 如何測試 omniauth 相關的 code
- 最後是選用 library 的時候遇到的各種考量


## Hello OAuth

[OAuth](TODO:LINK) 是一套 authorization 的標準。簡單的說，就是定義了一組規則讓所有的 application/developer 可以比較容易 follow。
它想要解決的問題大致上是這樣：

好比說我做了一個網站，目前有一些 users 在使用。我希望我的網站可以拿到這些 user 的 Facebook 大頭照、名字、和朋友清單。
這時候 OAuth 提供的解決方法概念上是這樣做的：

1. 先在 Facebook 上面申請一個 application
2. Facebook 用上面申請的 application 的名義，向 user 要求某些權限
3. user 同意之後，Facebook 則會發出一個 access token 給後我們的網站。
4. 從此之後，我們的網站就可以使用這個 access token，去向 Facebook 拿剛才提到的權限的資訊。

**這邊要注意的是，上面的 2、3 兩個步驟都是 Facebook 和 user 之間的互動。**也就是說，即使 user 不信任我們的網站，但他可以信任 Facebook 這樣的平台。所以這樣的作法某種程度上也可以解決 user 對小網站不信任的問題。

另外在上面的步驟 3，Facebook 要怎麼發 token 給我們的網站，隨著各種情境的不同會有很多[複雜的變化](https://www.digitalocean.com/community/tutorials/an-introduction-to-oauth-2#authorization-grant)，但概念上**最後的目標就是我們的網站要得到一個 access token。**

在拿到 access token 之後，OAuth 的流程就大致上結束了([註1](TODO:LINK-refersh token))。

如果今天我們要做的東西是 Social Login，我們很有可能會要支援除了 Facebook 之外的很多平台。每個平台都各自有自己的 document 和 API 格式。
而對於 Social Login 來說，application 最後要用 access token 在各個平台上拿到的資訊多半是大同小異。這時候選擇一個 library 來整合這一切感覺就相當合理了。

## Hello Omniauth

- abstract layer and role
- exposed interface
- omniauth structure and interface between strategies (TODO: read source)
- strategy structure
- testing
- design concerns
