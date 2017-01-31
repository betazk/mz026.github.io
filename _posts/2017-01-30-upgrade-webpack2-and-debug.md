---
layout: post
title: 升級 Webpack2 與 debug 大作戰
comments: true
description: "今天把 universal-redux-tempalte 的 webpack 升級到 webpack2。(在等了一百年之後，webpack2 終於有 stable release 惹！)。
升級的路上有踩到一些小雷，也經歷了一小段 debug 之旅。在這篇文章裡面，想把遇到的問題本身和 debug 的流程紀錄下來。"
tags:
  - react
  - webpack
  - debug
  - "universal-rendering"
---

今天把 [universal-redux-tempalte](https://github.com/mz026/universal-redux-template) 的 webpack 升級到 webpack2。(在等了一百年之後，webpack2 終於有 stable release 惹！)。
升級的路上有踩到一些小雷，也經歷了一小段 debug 之旅。在這篇文章裡面，想把遇到的問題本身和 debug 的流程紀錄下來。

## TL;DR

- Follow 一下官方的 [upgrade guide](https://webpack.js.org/guides/migrating/), 以我的 case 來說，主要其實只有 `resolve`, `loader` 的選項要修改。
- 用 `DefinePlugin` inject `process.env` 的時候，遇到 React 會壞掉([ref](https://github.com/webpack/webpack/issues/4000)), 用 `EnvironmentPlugin` 吧！
- 可以參考一下 [debug 之路](#section-1)

## 情境介紹

[universal-redux-tempalte](https://github.com/mz026/universal-redux-template) 是我做的一個 redux universal rendering 的 starter kit。最近嘗試把它用的 webpack 升級到 2.2。
它的架構大致上可以粗分為：

- [client-side](https://github.com/mz026/universal-redux-template/blob/798db78/app/app.js) webapp
- [server-side binding](https://github.com/mz026/universal-redux-template/blob/798db78/app/server/server.js)，是 universal rendering 的核心
- [build](https://github.com/mz026/universal-redux-template/blob/798db78/webpack.config.js) [process](https://github.com/mz026/universal-redux-template/blob/798db78/gulpfile.js)，主要是 webpack + gulp

細節的介紹可以參考 [README](https://github.com/mz026/universal-redux-template), 或者是[我之前寫的文章](http://mz026.logdown.com/posts/308147-hello-redux-2-3-server-rendering)（或[英文版](https://www.codementor.io/reactjs/tutorial/redux-server-rendering-react-router-universal-web-app) XD）


## Upgrade 的過程

在 follow 了官方的 [upgrade guide](https://webpack.js.org/guides/migrating/)，改了 `resolve`, `loader` 之後，webpack 可以順利把 code build 起來了。Server render 的部份看起來也是有正常運作。

但是卻在 browser 出現了以下的 error：

![error](https://cloud.githubusercontent.com/assets/680900/22277705/a7bd8632-e2f8-11e6-92cb-258cc1a3b8be.png)

```
Uncaught SyntaxError: Unexpected Token .
```

點開對應的行數，發現是出現在 React 的 code 裡面。
直覺懷疑是 config 有什麼地方有誤會（畢竟是在 upgrade webpack嘛）

在一番 google 卻都沒什麼進展之後，**想說把程式簡化到最基本的狀態看看能不能找到問題的癥結點。**

這邊大概是整個 debug 的方向

整個 app 可以分成

- server rendering (和 webpack 無關)
- webpack: client side 的 code 是由這邊build出來的，下面有
  - babel loader
  - 一些其他的 plugin


首先我想先確定，問題的點是不是真的和 server render 無關，

1) 於是直接用 `webpack-dev-server` serve 一個純前端的 app, 發現問題還是在。所以先不管 server rendering 了。

2) 於是我把程式的進入點[`app.js`](https://github.com/mz026/universal-redux-template/blob/798db78/app/app.js) 的東西全部 comment 掉，發現可以work！
於是發現，問題的點應該不單純只是 config 的關係，而是和 application 內的內容有關。

3) 接著，把進入點的東西慢慢地加回來。既然出現 error 的行數是在 react 的 code 裡面，那先 import 個 react 好了。
結果果然出現了同樣的錯誤訊息。

這時候問題就變成了：**用 webpack2, babel 建一個最單純的app，但是在 import react 的時候會出現 syntax error**


這個問題就相對單純很多了，因為一定有很多用 webpack2 建立的 react app sample。於是我立馬 [google 一個](https://github.com/ModusCreateOrg/budgeting-sample-app-webpack2)，

4) 然後把它的 webpack config 找出來換上去，結果發現可以成功 import react。

5) 接著，就把 webpack config 的 key 一個一個換掉，終於找到了是和 `DefinePlugin` 有關。

接著，在把 [DefinePlugin 換成 EnvironmentPlugin](https://github.com/mz026/universal-redux-template/commit/214fe5a) 之後，一切就恢復正了常了，耶！

## Debug 的流程

在 debug 的時候，如果可以找到問題的點，要解決起來通常會單純很多。
或者說，真正困難的部份往往是在 "找到問題的點" 這個步驟上面，而不是在後續的解法上面。
但找到問題的點通常並不容易，因為大部份在遇到問題的時候，程式已經很複雜了。
要根據 error message 或者是一些外在的症狀，在茫茫 code 海之中找到出問題的那一行，是一個漫長的過程。

藉由這次 upgrade webpack 的過程，綜合一些以前的經驗，把一些方向上的作法整理起來。
其實這些東西很可能大家在 debug 的時候都不知不覺在使用了。

在所有的步驟之前，最重要的一步是 **儘可能在各個時候加速 feedback 的速度**。

在 debug 的時候，簡單的說我們做的事情會是：

1. 做出一些改變
2. 看看 bug 是不是不見了，或者是不是還在

要在各個時候加速這個循環。好比說，如果某一個 bug 是出現在某一個 redux 的 reducer 上面，那與其用 browser + 手動按按鈕來重現，如果可以在 unit test 裡面，或是用 redux 的 dev tool 重現它，就會快很多。又或者是如果一個 bug 會出現在 user flow 很深的地方，例如說要在 "登入" →  "按某個 button" → "在某個 input box 輸入某個值"，那如果我們可以先把程式小改一下，讓每次 reload 就直接停在這個狀態，這樣的話也可以加快 feedback 的速度。

可能會有一百種方法可以加快 feedback 的流程，但要記得隨時問自己：還有更快的方法可以加速這個循環嗎？

在加快循環之後，接著就是 "怎麼找到問題的點" 的方法了。

### 大方向會是：

- 找到問題的大的區塊，先確認和哪些部份無關（像是把 server rendering 刪掉）
- 建立最單純卻可以重現 bug 的狀態（什麼都不做，只 import React）
- 在這樣的狀態之下，找到會 work 的方法（用 google 來的 webpack config 是可以運作的）
- 比較前兩步去找到問題的點

在上面的過程當中，有很多簡化程式的步驟，其實同時也加速了 feedback 的速度。
例如說，本來要重現 bug 的過程會是：

- 改動 webpack config
- 把 node process kill 掉重開，這個步驟包括了 gulp，webpack，node server

但在確定這個 issue 和 server render 無關之後，就可以直接跳過 gulp，node server 的步驟，直接讓 webpack-dev-server build/serve 一個靜態的 hmtl 讓我們跑 javascript。

## 所以 DefinePlugin 到底是怎麼了呢？

在一翻纏鬥之後，發現這是 [webpack2 的一個 bug](https://github.com/webpack/webpack/issues/4000)。 `DefinePlugin` 在轉譯的過程中，會試圖把要定義的參數轉成一個 object literal。

例如說我們如果用 DefinePlugin 定義了：

```javascript
new DefinePlugin({
  "process.env": JSON.stringify({"NODE_ENV": "development"})
});
```

並且我們的程式裡面會去 access `process.env` 的其他 key，像是

```
process.env.DEBUG = true
```

而 webpack 在 build module 的時候，用的是 `eval`，所以上面的定義就會被轉成：

```javascript
eval("{\"NODE_ENV\": \"development\"}.DEBUG = true")
```

然後我們就得到 syntax error 了。
正確的行為應該是要轉成：（多了一個 `()`）

```javascript
eval("({\"NODE_ENV\": \"development\"}).DEBUG = true")
```

這個 issue 看 [github 上面的狀況](https://github.com/webpack/webpack/pull/4010)應該是已經 merge 了，但不知道什麼時候會被 release 出來就是了。
也就是說，理論上在下一版 webpack 這個 issue 應該就不會再出現了才對。


## 後記

我覺得在大部份的時候，**遇到 bug 的解決過程通常會比 bug 本身還重要**。
像這次的這個例子，可能過幾天 webpack issue 的 pull request 就 release 了，所以可能再也不會遇到同樣的問題。
但是 "找到問題的策略和技巧" 卻是會一再地被需要。因為只要是有 code 就會有 bug，有 bug 就要知道發生什麼事，這件事情應該是很確定的。
沒有人可以 "教你 debug"，因為每個 bug 都不一樣。但是一些技巧和邏輯分析的能力卻是可以練習的，我想這應該是為什麼有人可以通靈 debug 的原因吧。
~~好想成為通靈debug王~~

覺得試著把這些東西記錄下來好像很不錯。
