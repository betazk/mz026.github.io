---
layout: post
title: React Universal Rendering Server 掛了怎麼辦？
comments: true
description: "前幾個星期，Codementor 的網站突然在半夜離奇死亡。這篇文章想要分享的是找到問題的過程和解決的方法。主要是針對 React universal rendering 的 NodeJS server 的部份"
tags:
  - react
  - universal-rendering
  - nodejs
  - performance
---

故事要從前幾個星期 Codementor 的網站斷斷續續地在台灣時間的半夜掛掉說起…
在半夜連續收到各個 channel 傳來 ping 不到網站的訊息之後，我們花了一番工夫把問題找到、解決。並且做了一些事情減少類似的事情發生的機率。
在這篇文章裡面，我想把尋找問題的過程和解決的方法記錄下來和大家分享。
主要是針對做 universal rendering 的 NodeJS server。

## 情境介紹

在 Codementor，我們的網站有很大一部份是使用 [React universal rendering](https://www.codementor.io/mz026/server-side-rendering-with-redux-and-react-router-8s8en3o7p) 來做的。所以會有一個 NodeJS 的server 負責發 request 給 API server 來做 universal rendering。
Server 是 host 在 Heroku 上面，並且有裝 Newrelic 來做效能的追蹤。

## 發現問題

由於 Codementor 的整個服務是分散在各個不同獨立的 services 之中，我們有在 Newrelic 上面建立一個 [insight dashboard](https://newrelic.com/insights)，
可以在一個畫面之內同時看到不同 server 的 Apex score, through put, transaction time 等等的資訊。
在接到 Newrelic 傳來 ping 不到 server 的訊息之後，我們立刻打開 insight dashboard 來看，
發現在那段時間之內，

- 各個 API server 的表現都是正常的，唯獨只有 universal rendering 的 server 出現了嚴重的 timeout。
- 即使 universal rendering 的 server 出現了 timeout，但當時的 throughput 並沒有明顯的高峰。所以也初步排除是因為整體流量太大導致 server 吃不消。
- 在發生 timeout 的時間附近，response time 的曲線是相當陡峭的。也就是說，問題應該不是經由累積而來，而是某一個或一些 request 突然造成的 timeout。
- 於是我們開了 heroku 的 metrics 來，發現那段時間之內，universal rendering server 的 memory 也狂噴。
- 狀況第一次發生的時間是在半夜，那時候並沒有任何人 deploy 新的 code，所以初步排除是"某個剛推的新東西造成的結果"。這也就是說，把 deploy rollback 在這時候應該是無法解決問題。


## 急救章

雖然說第一次發生 server down 的時間是在半夜，但是很幸運(?)的是，在開始上班之後又陸續發生了不少次。
這時候**在真正解決問題之前，我們希望先讓狀況先穩定下來以把使用者受到的影響降到最低。**
因為數據顯示，在發生 server timeout 的時候，都有 memory 爆衝的狀況，於是我們把 Heroku 上面的機器開到最大。然後開始 debug。

但是事情沒那麼容易阿！
即使我們把機器開到最大，但 timeout 的狀況還是持續出現 QQ。這時候 memory 並沒有吃光。

但雖然 scale up 無法解決問題(臉都綠了阿阿阿阿)，但這也告訴我們這個問題並不是機器資源的問題。
也就是說，應該不是 memory leak 之類的因素造成的。

## 開始 debug

這時候，我們的方向鎖定在找到 "是不是有某一種頁面(route)對應到的程式沒有寫好" 所以導致 server 壞掉呢？

如果是的話，那麼在發生問題的當下，**勢必有某一種 route 在那個時候被呼叫的次數/比例多於其他的時間區間。**。
這時候我們發現，我們好像沒有一個簡單的方法可以找到這樣的 route。
但是由於問題發生的時候是很突然的，所以我們可以很容易的找到在問題的時間點附近的"所有 routes"。

接著，我們把上面的各種 routes 各找幾個 request：好比說時間之內有 user profile 頁面、文章頁面被呼叫到。那我們就選某幾個人的 profile, 某幾篇文章的頁面。然後把這些 request 用同樣的 through put 打在 staging 的機器上。

但很可惜，這時候 staging 的器機都活得好好的，我們無法重現當時的問題 QQ

## 天眼通

折騰了一天之後，正當我們垂頭喪氣準備回家洗洗睡的時候，突然強者同事發現了可以重現問題的關鍵頁面。是某個特定文章的頁面！
只要一 request 到那個頁面，server 就會立刻 timeout，並且連帶其他後續的 request 也回不來了。
然後我們立刻問他說是怎麼發現這個神秘的頁面的，~~他說不要問，很可怕 XD~~
答案是他找了我們在 [papertrail](https://papertrailapp.com/) 上面的 heroku request log, 發現**那個 url 在某個特定的時間之後，就再也沒有活著回來過了**(這是什麼鬼故事的起手勢)。

在可以重現問題之後，接下來的情就簡單多了。我們發現因為我們會在 universal rendering 的時候去把 markdown 轉成 html，在這一步的時候發生了一點誤會噴出了無窮迴圈。又因為 nodejs 是 single thread，所以這時候其他的 request 也不用玩了，於是整個 server 就掛了。


## 賽後檢討

在把這個特定的問題解決掉之後，我們認真地進行了賽後檢討，歸納出一些要改進的項目：

### 1. 限制爆炸的範圍

以這次的狀況來說，是*一個單一的 bug 讓整個服務死亡*。這點是無法被接受的事情。因為 **bug 總是會有，但必須要讓 bug 發生的時候，盡可能地限制它的爆炸範圍 。**

針對這點，在 NodeJS 的環境下，我們目前沒有找到一個簡單的方法讓我們可以在某個 request 跑到無窮迴圈的時候設定 timeout 把它停掉。
但是倒是順路發現了一些本來應該做但是沒有做的事情們：

#### NodeJS cluster

在看了 Heroku 的 [document](https://devcenter.heroku.com/articles/node-concurrency)，我們終於把 NodeJS 的 app 加上 [cluster](https://nodejs.org/api/cluster.html)，讓它可以跑多個 process，充份利用機器上面的資源。
對於"限制爆炸範圍"來說，這是一個治標的作法。意思是說，本來是只要一個 request 產生無窮迴圈就可以讓 server 死亡。現在如果有多個 process 同時在進行，利用不同的 CPU 進行運算，理論上要所有的 CPU 上的 process 同時產生無窮迴圈才會讓服務整個死掉。
當然，在有一定 throughput 的 web server 的情境下，這樣的作法產生的幫助幾乎是微乎其微。但跑在 cluster 上面可以充份利用資源這件事就足夠讓我們加上去了。

另外，在這點上面還有一些出乎意料的小發現。就是**在同樣的機器之下，加上 clustering 並不一定會讓效能提升**。

以我們的例子來說，在把無窮迴圈的bug解掉之後，我們的機器還是先停留在 heroku 的最高的等級(performance-l)。在加上 clustering 之後, 整體的速度反而變慢了。(不負責任)猜測是因為在很高的運算能力和 request 的 loading 不太大的狀況下，做 context switch 的 overhead 反而變成主要的瓶頸。
但後來我們的就把機器 scale down 了，於是這時候加上 clustering 的效能就比沒加的時候好了。

### 2. 讓 bug 更容易被重現

以這次的例子來說，我們等於是用天眼通找到 bug 的。但是天眼通可不是天天都能通到的。
我們需要一個更有系統的方式讓我們找到有問題的 request。

我們可以把 request 分成兩個階層：

- 第一層是用 route 的類型來分。如果是 `react-router` 的話，它就會是上面的 path pattern。好比說我們會有一個 route 是 `/:username` 代表使用者的 profile page(像[這個](https://www.codementor.io/mz026))，而另一個 route 是 `/:username/:article_id` 代表某個文章的頁面(像[這個](https://www.codementor.io/mz026/server-side-rendering-with-redux-and-react-router-8s8en3o7p))
- 第二層則是 *同一個 route 下的不同頁面*。好比說同樣是 user profile 的頁面，可能會有很多不同的 user。像是 `/mz026` 和 `/mz027`。

以我們的情境來說，在 Newrelic 上面，我們可以簡單的發現發生在第一層上面的問題，但第二層的問題就不是那麼容易在上面被找到。
也就是說，如果整體來說 article 的頁面很慢，那我們可以很容易地從 Newrelic 上面發現。
但如果大部份的 article 都沒問題，只有特定幾篇表現很差，那從 Newrelic 上面就不好發現了。

像這類的情境，我們初步討論的結果是，如果可以把 request 的 log 丟到一個可以有效 query 的環境 (例如說 elasticsearch) 應該會很不錯。這樣的話，我們就可以有效率地回答在 [天眼通](#section-3) 的時候發現的 "有沒有哪些 request 是在某一個時間點之後，就再也沒有活著回來了？" 這個問題。

### 3. Universal Rendering

大絕招天天放也是會累的。
在認真檢查之後，我們發現我們好像不小心在一些其實不需要 SEO 的頁面(像 dashboard) 也不小心做了 universal rendering。
但這樣其實會增加 server 不必要的負擔，所以把這些地方的 universal rendering 移除掉。

另外我們也懷疑文章的頁面在 server 上面載 markdown 轉成 html 會不會造成很大的負擔呢？
如果會的話，可能就要引進一些 cache 機制來解決。
於是我們把markdown 轉 html 的地方先暫時 hard-code 成一個簡單的結果，然後丟到 staging 上面用同樣的 request pattern 和 throughput 試看看。
結果發現影響並不大，所以就先保持原狀。


## 結果

實際上，我們除了修掉 bug 之外，真正做的事情只有

- 把 nodejs 加上 clustering
- 把可以不用 universal rendering 的地方移掉

目前結果是，universal rendering 的 server 平均比之前快了 35% 左右。


## 心得與感想

- 有好的 alert 機制很重要。當 server down 的時候，被連續的 notificaiton email 叫醒總比隔天被 user 的信罵醒還好。但 alert 的機制可大可小，我認為找到"符合目前需求的最簡單作法"也是重要的課題。
- 有好的 profiling tool/dashboard 很重要。在預算允許的狀況下，Newrelic 很不錯。但這個東西也是可大可小，同上，找到"符合目前需求的最簡單作法"也是重要的課題。
- 關於 performance 的調校，我很同意[這篇 hackers news](https://news.ycombinator.com/item?id=14709076) 說的：等到遇到問題了再來解決。並且設定一個效能的目標，當達到了該目標後就停下來。(Don't, unless you've built it and it's too slow for users. Have performance targets for how much you need to improve, and stop when you hit them.) 因為效能的調校是無窮無盡的，並且時常要在 code quality 或者是結構上做出某種取捨。
- 在要調整任何部份的效能之前，先想辦法 profile。就算是粗略的 profile 也比沒有好。常常會得到意外的結果。
