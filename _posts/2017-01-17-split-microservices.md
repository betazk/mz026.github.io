---
layout: post
title: 如何切割 Microservices
comments: true
description: "這是一篇經驗分享的文章，內容是關於把一部份的功能從一個 monolithic applicatoin 拆出來變成獨立的 service 的過程。主要會介紹切割 service 的考量，還有過程當中遇到的問題們"
tags:
  - microservice
  - backend
  - architecture
---

這是一篇經驗分享的文章，內容是關於把一部份的功能從一個 monolithic applicatoin 拆出來變成獨立的 service 的過程。
主要會介紹切割 service 的考量，還有過程當中遇到的問題們。希望可以給有同樣情境的人們一點幫助 :)


## Hello Microservice

在一切開始之前，首先要簡介一下 microservice。

Microservice 在 [wiki](https://en.wikipedia.org/wiki/Microservices) 上的定義是這樣的：

> Microservices is a specialisation of an implementation approach for service-oriented architectures (SOA) used to build flexible, independently deployable software systems.

翻成白話文大概是說，把整個服務的功能分成若干個互相獨立的小塊（service），讓這些 service 間以 API 互相溝通以達成整個服務要做的事情。和 microservice 相對的就是 monolithic application，指的就是把所有的功能集中在一起的 application。

把服務拆成小塊有什麼好處呢？

**在回答這個問題之前，應該先問的是：不拆開有什麼壞處呢？**

如果回答不出來，那太棒了！代表你的程式現在的狀況可能不適合拆成小的 service。

這邊大概列出一些 microservice 可以解決的問題：

當程式越來越大的時候…

- deploy 越來越慢，但又因為所有功能都和這份 code 有關，所以總是要一直 deploy 全部的程式
- 程式內部的介面是 class、method 等。當 devleoper 變多的時候，要清楚溝通這所有的介面變得越來越難。
- 可能有一部份的功能很適合一種技術（像是 relational database），但另一部份的功能卻適合另一種（像 nosql）

像是上述的問題，可能拆成小的 service 就可以解決。但當然，microservice 也不是萬靈藥。下面列出一些可能會遇到的問題：

- 不同的 service 之間要互相溝通，本來可能是 method call 的，現在變成了要跨過 network（像是 http request）。
- 因為 network 有著不穩定的天性，所以要做更多的 error handling
- 程式的 monitoring，像是 performance 等，要做得更完整。不然可能會有其中一個 service 不小心死掉了也沒人知道。

**總之，microservice 不是在所有狀況下都適合的解法。並且大部份的時候會是一個漸進的過程：就是從一個 monolithic application 慢慢把一些東西拆出來。**



## 情境簡介

最近公司的產品要規畫蠻多和 community 相關的功能。當初規畫的時候就有預期它可能有一天會變成一個獨立的 service。

這部份的功能主要又分成三大塊，每一大塊各自有若干不同的 table 在存放相對應的資料。
三大塊之間，在有些時候也會需要互相溝通。


## 要拆成獨立的 service 的原因

大部份的時候，要把一部份的功能拆出來變成獨立的 service 是會有一些原因的。以這個 case，我們考量的點是：

### 1. 這部份程式的行為是相對獨立的

在規畫這些功能的時候，我們可以預想到它有各式各樣的發展可能。但整體的大方向是和本來現有的行為沒有直接關係。
這樣的狀況以功能面來說，是很適合自己獨立出來的。

### 2. 即將快速發展

可以預見的是，這塊功能會快速地發展，並且會有各式各樣的 iteration 以驗証我們的各種假設。
這意味著，如果太慢才拆出來的話，就會要一次搬動很多 code XD。
~~正所謂有些事情現在不做，一輩子都不會做了。就是這種感覺 ?~~

### 3. scope 大 + legacy

我想有一大部份促成拆解 monolithic application 的原因是 [legacy code](/20161210/working-happily-with-legacy-code)。雖然說來有點慚愧，但其實就是在沒有任何包伏的狀況下寫新的功能其實還蠻快速的。



## 在搬家之前

雖然說有預期要把它拆成獨立的 service，但是並沒有在一開始的時候就直接建在另一個 service 上。
主要原因是因為在初期驗証 idea 的階段，
希望可以越快上線越好，所以放在本來的 service 上面可以省去建立一整個 infrastructure 的時間。
(往壞處想，如果驗証 idea 失敗了，那也省去了不少力氣 XD)

*在完整驗証 idea* v.s. *在程式長得太大之前趕快搬家* 這兩個因素之間取得了某個時間上的平衡點之後，就可以開始動工了。

在這個階段，我們對於搬家有一些基本的要求：

### 1. 必須是漸進式的搬動

意思是說，我們希望**整個搬家的過程不要是整個都搬完然後再一次上線，而是可以找到某種方法，讓我們可以一次搬一部份就可以立刻 relase 到 production。**

這樣的用意在於減少潛在的風險。因為如果我們花了兩三個星期搬動的努力，在最後一刻上線才發現哭哭不work，這樣的狀況下 user 一定會受到很大的影響。

**再者以人力的規畫和時程的預測來說，把 relase 時程切小塊，有助於我們提早發現一些"要relase才知道的問題"。因為像這樣的搬動過程，即使有仔細的規畫，還是充滿了不確定性。在最糟的狀況下，如果我們relase第一個milestone 之後發現這一切超難搞，還來得及在付出太多時間之前想別的辦法。**

### 2. 在 relase 的時候，不能有 downtime

我們希望找到某種方式，讓我們在切換新/舊 service 的時候，user 不要有任何感覺。或者就算有感覺，也不要太強烈XD。至少整個服務*大致上*是可以使用的。

這部份的原因應該就很單純了，因為如果可以的話，誰想要有 downtime 呢？


## 搬家的過程

搬 code 的流程，其實說來單純。下面的新/舊 server，code 其實指得是兩個不同的 server（甚至是不同的 sub-domain）/ 兩個不同的 repo。
兩個 server 各控制自己的 database。

過程大致上就是：

1. 把新的 server 架好。
2. 定義出一個要搬動的 milestone。
3. 把新的 code 寫好，deploy 到新的 server上去。
4. 把 client 寫入的 endpoint 指到新的 server 上去。
5. 想辦法把舊的 database 的 data 倒到新的 database。
6. 把 client 讀取的 endpoint 指到新的 server 上去。
7. 完成一個 milestone 的搬家惹，耶！


在上面的流程裡面，比較複雜的會是 4～6這三步。
因為在搬家的同時，舊的 server 也還在持續運作，不斷地有 user 在寫入新的 data。
**這時候我們希望在沒有 downtime的狀況之下，讓user寫入的data 不要不見。所以用犧牲一小段時間的讀取的精準度來做到。**

也就是說，在步驟 4 完成之後，user 寫入的 data 會寫到新的 database，但讀取的 data 卻是舊的。也就是說，在這時候他會看不到他剛才寫入的 data。
這時候我們把舊 db 的 data 倒到新的 database，然後再讓 client 去讀新的 db。這時候他剛才寫入的 data 就又出現了。

**這樣的作法之下，"讀取不到剛才寫入的東西"的時間，取決於第 5 步驟，也就是移動 data 的時間。在 data 還只有少少的狀況下是可以接受的，但是如果data 太多導致要搬動 data 會花很久的時間的話，可能就要再想更厲害的作法了。**

在完成一個 milestone 的搬動之後，接下來要補足各個 milestone 的功能彼此之間互相溝通的 API。
以我們的例子來說，說本來在舊的 server 上面有 newsfeed 和 notification 的功能。在 newsfeed 這邊發生了某些事情之後，要送出notification。
在搬家之前，是直接用 method call 做到這件事情。
但如果我們先把 Newsfeed 搬到新的 server 上面，而 Notification 是屬於下一個 milestone 才要搬家的功能，那麼 "發 notification" 這件事情，在 Notification 搬好之前，就要透過 API call 來完成了。


## 撇步們

上面大致上就是搬家的流程，下面我把一些過程中有遇到的問題和小撇步們整理起來：

### 好好找 service 之間的界線

我認為這是最重要的一點。

在決定 "什麼東西要切成一個 service" ，和上面提到的 "怎麼決定哪些東西要放進一個搬家的 milestone" 這些事情上面，其實沒有一個絕對的標準答案。
**但是如果這備界線切得不好，就會出現不同的 service 之前相當頻繁的溝通，整體來說反而會讓整個系統變得更複雜並且難以維護。**

**要定義出精準的界線，首先必須對整個服務的架構有很深入的了解。**
下面是一些我認為可以參考的點：

- Service 的界線通常是跟著 business logic, 而不是技術上限制來定義的。也就是說，我們可能會看到 "付費的 service", "推荐系統 service"，但不應該看到 "NoSQL service"，或是 "NodeJS service"。
- 屬於同一個 service 的程式，彼此之間會有很密切的溝通。而這一個 service 本身，是可以定義出一些 API 讓外部其他的 service 來使用。
- 一個 service 會直接掌握它對應的 data，這些 data 對於其他外部的 service 來說是一個只能透過 API 來存取的黑盒子。
- 可以參考 [Bounded Context](http://softwareengineering.stackexchange.com/questions/237513/what-in-reference-to-ddd-is-a-bounded-context?answertab=active#tab-top)
- 也可以參考 [Conway's Law](https://en.wikipedia.org/wiki/Conway's_law)

但說到底，這是一個很難的問題。這也是為什麼大部份的時候，我們會希望不要在一開始 business logic 都還不確定的時候就先畫出界線定義出各個 microservice。反而是先集中寫在同一個地方，等整個 domain 的運作都比較穩定之後再切開。


### 從 API 切入

在決定好界線之後，我們要怎麼知道有哪些東西是需要被搬家的呢？
也就是說，我們要怎麼確定沒有漏搬哪一個 table 或是 controller 呢？
這個問題我們解決的方法是 **從API切入**。就是說，我們先找到哪些 API 是要被搬家的，這樣一來只要這些 API 都在新的 server 上出現，我們就知道大功告成了。


### [Strangler Pattern](https://www.martinfowler.com/bliki/StranglerApplication.html)

Strangler Pattern 是在 refactor 架構的時候常見的作法之一。
在一開始的時候，其實有考慮過用 strangler pattern 來做。如果是這樣的話，做法就會是：

- 先做一個新的 server
- 把所有對應的 API endpoint 全部都 proxy 到舊的 server 上面去
- 讓 client 指到新的 server
- 再一個一個把功能換掉

後來沒有採用這樣的做法的主因是，因為目前來說我們的 client 只有一個網站，並且是由同一組 team 來維護的。
也就是說，對於 client 來說，要換 endpoint 相當的快。所以好像就不太需要特別先做一個 proxy了。


### 定義出搬家的 milestone

大部份的時候，我們會希望可以先搬一部份就立刻 release，而不是一次積很多再一次出清(?)。
主要的原因是，像是這類型的 architecture refactor，很常會有 integration 的問題。
也就是前面提到的"要relase才會發現" 的問題。
以時程來說，切小塊 relase 可以更精準的掌握整體的時程。另一方面，以 debug 的角度來說，如果真的出現了什麼可怕的問題，
當搬動的 code 相對少的時候，要修起來也比較單純。

但是要分階段搬家，勢必要面對 "每一次要搬動哪些" 這個問題。

**我認為這個問題其實是縮小版的 "service 的界線"。
理想上，我們會以一組彼此間互動性很高的 feature 為單位來搬動。**

但常常要搬家的原因之一，是因為本來的舊 code 沒有明確的把這些界線定義出來，又或者是邏輯上有，但是在 code 上面是整個 couple 在一起的。像這樣找不到明確的 "彼此間互動性很高" 的 feature 界線的時候，另一個作法是退而求其次，一次搬動一個 table。

這種時候，有一些小技巧可以參考：

當要搬動一個 table 和它對應的程式的時候：

- 一樣以 API 為出發點
- 先從 creation 的 API 開始搬動
- 如果 table 之間有從屬關係，先從邏輯上的 parent model 開始搬動。

但是如果是退到這樣的搬動方式的話，在每一次 release 之間，會有比較多 "因為搬動而產生的新 API" 要另外做上去。
例如說，假設我們本來的程式的行為是 "如果 user 發表了一個 post，要產生 notification 通知 subscriber"。
如果我們第一個 release 是先把 `Post` 的 creation 搬到新的 service，那麼當 user 建立了一個 post 之後，要想辦法讓舊的 server 也產生一樣的 record 來讓那些 "還沒有搬家，但是會和 post 有關" 的行為正常運作。

大致上來說，如果每次搬動的程式和舊的 service 牽扯越深，會有越多像上述的行為要處理。

**選擇每次的搬動範圍，其實是在 "release 的風險"和這些"因為搬動而產生的新 API" 之間作一個取捨。
但是很明確的是，如果一開始的程式架構越整齊，這些取捨會越簡單。**


### 注意 table id sequence

在把舊的 database 的 data 倒到新的 database 之後，如果 table 的 id 是用 auto increment 的sequence來做的話，
要注意sequence重覆的問題。
假設我們使用的 database 是 Postgres, 然後本來有一個 post table，它的 id default 值是

```
nextval('posts_id_seq'::regclass)
```

這時候如果舊的 database 有 `id` 0~100 的 records，那麼當倒到新的 database之後，要記得把新 db 的 sequence 設成大於 100 的值，不然當倒完 data 之後，再在新的 database 建立新 post 的時候，就可能會有 id 撞到的問題。


## 後記

在寫程式的過程當中，需求總是會一直演進的。這也就是為什麼我們總是一直需要不斷的 refactor 讓程式可以跟得上 business logic。Refactor 有可能發生在各種階層，從 code layer 到 architecture layer 都有可能。
我覺得以這次的過程來說，
感想是**程式碼的品質和完整的測試還是一切的基礎。**
如果 code 的架構清楚，測試完整，要搬起來都不會是什麼太難的事。但是假設這次的搬動後面沒有 unit test 來撐著的話……好，不要去想這種可怕的問題。這樣的狀況還真的不知道怎麼搬家ㄎㄎ。
