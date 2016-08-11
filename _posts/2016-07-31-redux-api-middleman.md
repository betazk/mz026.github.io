---
layout: post
title: Redux API Middleman
comments: true
description: "最近和同事們一起合作，做出了一個把 \"送 API request\" 這件事情抽象出來的 redux middleware: redux-api-middleman"
tags:
  - redux
  - react
---

最近和同事們一起合作，做出了一個把 "送 API request" 這件事情抽象出來的 redux middleware: [redux-api-middleman](https://github.com/CodementorIO/redux-api-middleman)

這個東西本來是在 [redux-universal-template](https://github.com/mz026/universal-redux-template) 裡面的一個 middleware，但隨著功能越加越多，再者有別的 project 也要用到一樣的東西，所以索性把它抽出來變成一個獨立的 [npm package](https://www.npmjs.com/package/redux-api-middleman)

<!--more-->

# Why?

為什麼要把”送 API request” 這件事情特別拿出來討論呢？主要是因為送 API 這件事情是在一般的 web app 裡面很常見的非同步(asynchronous) 的行為。而 asynchronous 的行為本身實作起來就比較複雜一點，要寫相對應的測試更是複雜。如果可以有某種方式讓我們把件事情抽象出來讓它單純一點，那就太好惹！
Redux 在這方面提供了一個很好的切入點，讓我們可以把這當中需要的東西抽出來變成一個 middleware。

# What?

如果要把 "送API request" 這件事情抽象出來的話，要把什麼東西抽出來呢？或更精確地說，我們希望抽出來之後，使用起來是什麼樣子呢？

## Asynchronous to Synchronous

首先，我們希望送 API request 這件事情從一個 asynchronous 的行為變成一個 “貌似 synchronous” 的行為。之所以說 “貌似” synchronous 的原因是，送 API 這件事情本質上就是 async 的，但我們可以透過某種抽象的過程，讓它寫起來和測試起來的時候，像是在處理一個 synchronous 的東西一樣。

## 支援 [Universal Rendering](http://mz026.logdown.com/posts/308147-hello-redux-2-3-server-rendering)

送 API 這件事情，其實和 React Universal Rendering 息息相關。為什麼呢？因為在做 universal rendering 的時候，web server 必須向 API server 拿要 render 的 data，而這部份的程式其實和前端是共用的。但當這件事情發生在 server 上的時候，我們會多出一個需求：就是在 server 上要知道什麼時候 data 才全部都拉下來了，這時候才可以把 React components render 出來送給前端。

## Side Effect Friendly

API request 常常伴隨著 application 的 side effect。好比說，在 login 的 request 成功之後，我們要把 store 裡面的 `currentUser.isLoggedIn` 更新成 `true` 之外，要把 user route 到另一個頁面。這時候 "route 到另一個頁面" 就稱之為這個 request 的 side effect。我們希望把 "送API request" 這件事抽象出來之後，可以讓我們有一個明確的地方來管理這類的 side effects。

## API 連發

在某些時候，API 的設計並不一定會和前端的使用流程密切的貼合。也就是說，在這個時候我們在 web app 這端要呈現出一個畫面或者完成一個行為的時候，有可能要連續送出一個以上互相有關聯的 API call。例如說，第一個 API 可能是

```
url: /me [GET]
returns:

{
  "id": 1,
  "blog_id": 1234
}
```

而第二個 API 是

```
url: /blogs/:blog_id [GET]
returns:

{
   blog details...
}
```

這時候如果真正改變我們 application state 的是第二個 request, 而要發出第二個 request 必須要等一個 request  回來之後才可以執行，這時候我們就會希望我們的抽象出來的東西可以讓我們輕鬆地做到這件事情。


## Replay

這個部份的功能就屬於在比較特殊的時機才會用到了。最近我們有一部份的 request authentication 從 session 轉換成用 [JWT](https://jwt.io/) [single sign on](https://en.wikipedia.org/wiki/Single_sign-on) 來做，大致上的出發點**是希望每一個 server 要做 request authentication 的時候可以不用去問一個統一的 data store** (像是 database 或 redis 等)。但同時我們又希望在必要的時候可以把一個 token   revoke 掉。於是在 client 這邊就必須要做到：

**如果一個 request 發出去，server 回傳 401 的話，這時候要先發一個 refresh token 的 request, 然後再把本來的那個 request 重發一次 (replay)。**

但在 client 這邊的實作上，同時又不希望這件有點複雜的事情讓整份 code 變得太過可怕。

## Parameter Mutation

另外我們還想要達成的事情是 parameter mutation。意思是說，在每個 request 自動帶上某些參數，比如說：

- 如果 store 裡面有 `currentUser.accessToken` 的話，就把它帶在 request 的 header 上面的 `x-access-token `這個 field
- 把每個 server 回來的 response 都 [`camelizeKeys`](https://github.com/domchristie/humps#converting-object-keys)


# How

綜合以上的各點需求，於是我們決定把這個東西用 redux middleware 的型式抽象出來。好吧，其實不是。最一開始其實我只是想做到 async to sync 和 universal rendering 而已，於是就把 [redux real word example 裡面的 api middleware ](https://github.com/reactjs/redux/blob/master/examples/real-world/middleware/api.js)拿來改，想不到後來功能越加越多才變成現在這個樣子 XD

而在實作方面，有一些有趣的地方想跟大家介紹一下。

## Async to Sync

這個部份是 [redux-api-middleman](https://github.com/CodementorIO/redux-api-middleman) 的最根本的部份。透過它，在 action creator 裡面要發一個 request 就會變成這樣：

```javascript
import { CALL_API } from 'redux-api-middleware'

export const GET_MY_INFO_SUCCESS = Symbol('GET_MY_INFO_SUCCESS')
export function getMyInfo () {
  return {
    [CALL_API]: {
      method: 'get',
      path: '/me',
      successType: GET_MY_INFO_SUCCESS
    }
  }
}
```

**這樣一來，發 request 給 server 這件事情在寫法上，就從本來的 async 的型式變成 return 一個帶有 `CALL_API` 這個 key 的 action。而真正發 request 出去的這件事情則在 middleware 裡面完成。**

這樣的做法可以同時讓實作和測試都變得相當單純。

## Universal Rendering

在做 universal rendering 的時候，要怎麼讓 server 這邊知道 request 都已經 ready 了可以準備 render component 了呢？
答案就是用 promise 來完成。大致上來說呢，**就是讓 middleware return 一個 promise，這樣一來在 `store.dispatch` 的時候就會拿到這個 promise。如此一來就可以知道 API call 什麼時候完成了。**

## Side Effect Friendly

## API 連發

連發API的作法，在 action creator 裡面會長成這樣：

```javascript
import { CALL_API, CHAIN_API } from 'redux-api-middleman'

export const GET_BLOG_INFO_SUCCESS = Symbol('GET_BLOG_INFO_SUCCESS')
export function getBlogInfo () {
  return {
    [CHAIN_API]: [
      () => {
        return {
          [CALL_API]: {
            method: 'get',
            path: '/me'
          }
        }
      },
      (meResponse) => {
        return {
          [CALL_API]: {
            method: 'get',
            path: `/blogs/${meResponse.blogId}`
            successType: GET_BLOG_INFO_SUCCESS
          }
        }
      }
    ]
  }
}
```

大致上來說，就是把每一個 request 得到的 response body 傳到下一個，讓下一個 request 可以根據上一個 request 的結果去做事情。

連發 API 這件事情其實是發一個 API 的延伸。也就是說，**我們可以把只發一個 request 的情形看成是連發的 special case** (就是連發但只有一發這樣)。
所以實際上在實作的時候，如果接到 `CALL_API`(只發一個) 這種 type 的 action，其實會把它轉成 `CHAIN_API`(連發) 的 type 然後再 `dispatch` 一次。

而在 [middleware 的實作](https://github.com/CodementorIO/redux-api-middleman/blob/master/src/index.js#L48-L52)上，是把每個 `CALL_API` 的內容包裝成一個 promise, 然後再把這一連串的 promise reduce 起來，最後再 return 出去讓 universal rendering 使用。


## Replay

像上面所說的，我們想要可以在 request 失敗的時候，用 config 的方式介入一個 interceptor。**在這個 interceptor 裡面，我們可以選擇要重發 request 或者不管它就讓它 pass 到 reducer 那邊去**。以我們的例子來說是這樣的：

- 如果 API call request 的 error code 是 401，這時候我們要另外發一個 refresh access token 的 request 給 server。在 refresh 成功之後，就重發一次本來的 API call request。
- 如果 API call request 的 error code 不是 401 的話，那我們就不要管它，就讓它 pass 到 reducer 那邊去。

上面的情形我們用 config `errorInterceptor` 的方式來完成：

```javascript
import apiMiddleware from 'redux-api-middleman'

apiMiddleware({
  baseUrl: 'https://api.myapp.com',
  errorInterceptor: ({ err, proceedError, replay, getState })=> {
    if(err.status === 401) {
      refreshAccessToken()
        .then(()=> { replay() })
    } else {
      proceedError()
    }
  }
})
```

這樣一來，我們就可以在其他的 action 不知道這件事情的狀況下，做到偷偷 refresh access token 的行為惹。

## Parameter Mutation

在發 API request 的時候，有時候我們會想要在每一個 request 上面多帶一點東西，或者是把一些東西轉成另一個格式。
例如說：

- 在每一個 request 裡面，把 access token 帶在 header 裡面
- 在每一個 request 裡面，把我們 application 送出去的 camel case 的 request body, query string 等轉換成 snake style
- 在每一個 response，把 server 回傳的 snake style 的 response 轉成 camel case

像這些東西要每一個 request 都分開做就太痛苦惹。
在 [redux-api-middleman](https://github.com/CodementorIO/redux-api-middleman) 裡面，我們把這類的東西抽象出來並且分成兩類：

- 全部 request 都要 apply 的，像是上面提到的 access token 的
- 依據每個 request 而改變的，像是 camel case / snake style 的轉換

以全部 request 都要用到的東西來說，我們可以在建立 middleware 的時候，加入一個 `generateDefaultParams` 的 function:

```
import apiMiddleware from 'redux-api-middleman'

apiMiddleware({
  baseUrl: 'https://api.myapp.com',
  generateDefaultParams: ({ getState })=> {
    return {
      headers: { 'X-Requested-From': 'my-killer-app' },
    }
  },
  maxReplayTimes: 5
})
```

這個 function 最多可以回傳三個 keys，分別是：

- `headers`
- `query`
- `body`

而在每一個 request 裡面，這三個回傳出來的 key 就會在每一個 API request 被當成 default 的參數被放進去了。

# 結語

發 API 這件事情是 web app 裡面最常做的事情之一，但它的 async 的特性也常常在實作或測試的時候令人頭大。
目前這樣的抽象方法在我們日常開發的production app 都還算OK。如果有任何想法都非常歡迎 [pull request](https://github.com/CodementorIO/redux-api-middleman) 阿！！
