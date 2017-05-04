---
layout: post
title: 快樂搬移 Legacy Schema
comments: true
description: ""
tags:
  - "legacy-code"
  - programming
---

這陣子，Codementor 的主要服務之一 "1-1 Live Session" 要做一個架構上的大調整。主要的原因是我們想要把不同的服務之間相互整合，
基本上就是要擴充原有的產品邏輯和行為，讓它變成一個更廣義的概念。而在程式上，自然就需要相對應的改動。
"1-1 Live Session" 這部份的程式碼是我們最早的產品之一，可想而知的是這部份要擴充起來一路上會要還掉不少技術債。

在搬動的過程中，我試圖歸納出一些類似 SOP 之類的作法，但後來發現這實在有點難，
因為每一個要搬動的功能，所遇到的問題和挑戰幾乎都不同。
但還是整理了一些概念性質和方向性的東西記錄下來。

## 情境介紹

其實這次的搬動目的很簡單：就是要把本來的 "Codementor For Hire" 整合進 "1-1 Live Session"。

我們的目的是希望讓使用者在貼出一個 "1-1 Live Session" 的問題的時候，和他要找一個 "For Hire" 的流程和體驗是類似的。
但當然這篇文章的目的不是介紹這兩者的細節。簡單地說，在程式上我們面對的狀況就是：

有一個現有的服務(1-1 Live Session)，它有相對應的 database schema, data, API, front-end code。
我們要把這個服務做一些商業邏輯上的調整，讓它可以滿足新的需求。
**其中這個新的需求是本來的延伸**，也就是說，本來有的行為還是要留著，但要加入新的行為。


## 邊界條件們

其實上面提到的情境是再平常不過的事情。就是要擴充本來程式的行為嘛。
但是當本來的程式不是在一個最健康的狀況的時候，本來簡單的事情就變得有點痛苦惹。
"1-1 Live Session" 是我們最早的程式碼之一，
隨著我們產品一路演進，這部份的程式從 database schema 到 application layer 的程式其實都和現有的使用情境有著相當的落差。
再加上早期的程式，很多是沒有 unit test 的，有一部份甚至在架構上也不是 API，而是以 form post 的情境存在著。
也就是說，光是要定義出 "現在的程式有哪些行為"、"使用者除了這個流程外，還有哪邊會 trigger 這個行為？"，就不是件容易的事情。

但是技術債總是要面對，在和同事們認真討論過後，大概把問題和解決的方向定義出來：

雖然說 application layer 的程式也是有很大的進步空間，但 **database schema 和現有的行為不一致才是問題的根源。**
但這次在功能上要擴充的部份，其實只有佔整個 "1-1 Live Session" 的一部份而已，
也就是說，我們如果把整個 "1-1 Live Session" 整個整理乾淨，會遠超過這次功能擴充的 scope。
~~換句話說就是可能要做一百年，不能承受阿阿阿 QQ~~

**所以我們針對這次要擴充的內容設計出符合現有使用情境的 database schema，
然後把這些功能做在新的 database schema 上面，並且把其他不用搬動的功能留在原地。**

TODO: graph


## 為什麼 legacy schema 很可怕？

> 如果說世界上有什麼東西比 legacy code 還要可怕的話，那一定是 legacy schema。

> 但如果說，世界上有什麼東西比 legacy schema 還要可怕的話，那就是 **legacy code + legacy schema**

Legacy schema 可怕的地方在於，application code 基本上都是圍繞著它建立的，
如果整份程式是根據一個不合時宜的資料結構設計的話，
那就會要在各個地方用很多特例或者是晦澀難懂的判斷來讓整個行為是正常的。
這樣一來在維護上自然就會造成很大的負擔。

所以如果有機會整理 [legacy issue](http://blog.mz026.rocks/20161210/working-happily-with-legacy-code) 的話，**legacy schema 必須死！**

## 目標與策略們

在下面的篇幅中，我想把一些我們這次搬動的策略和目標整理出來和大家分享。

### 目標：盡可能縮小每次 release 的 scope

在這次大的改動的過程中，我們發現要估計時程是有困難的。
一來是因為要搬動的內容本身就很多，二來也是因為這些東西有很多的 legacy 在裡面，
所以大多的時候是要做下去才會知道裡面有多可怕的東西要處理。

但顯然地，無法估計時程的工作是不能接受的。
這個時候我們退一步去想，估計時程的目的是什麼呢？（這根本一個人生的問題吧）

**估計時程的目的，有一大部份是為了讓其他的工作可以做相對應的安排。**

舉例來說，如果有工作 A 我要做三天，那麼另一個很重要的工作 B 就要第四天才能開始。

> 如果我們可以把一個無法準確估計時程的工作切成很多小塊，每一個小塊都是做完就要立刻 release 到 production 的那種。那麼這樣一來，人力的資源就會變得有彈性得多。

至於說"切成小塊"是要切到多小，則會依每個團隊和狀況而不同。以我們的例子，每一個小塊大約是一個人做一到兩天的量。

好比說本來這次的搬動我們預估這次的搬動需要一個人做三個星期，但一次估計三個星期的時程其實是很不準確的。
並且在這三個星期（如果有 delay 就更久惹）之內，如果有更緊急的事情發生，那這個人因為正在搬動搬到一半，
所以無法抽身來幫忙。

再者，因為搬動的東西因為還沒 release 到 production，
但一路上同一份 code 有其他的 feature 正在演進，所以這個搬動勢必要一路狂解 conflict，[很不方便](http://zh.pttpedia.wikia.com/wiki/%E5%BB%BA%E8%AD%B0%E5%8D%83%E8%90%AC%E5%88%A5%E9%80%99%E6%A8%A3%E5%81%9A%E2%80%A6%E4%B8%8A%E6%AC%A1%E6%9C%89%E4%BA%BA%E9%80%99%E6%A8%A3%E2%80%A6%E7%B5%90%E6%9E%9C%E2%80%A6%E5%B0%B1%E5%B0%91%E4%B8%80%E5%80%8B%E2%80%A6%E5%BE%88%E4%B8%8D%E6%96%B9%E4%BE%BF)。

但上面的狀況如果我們把三個星期的搬動，切成很多可以立刻 release 的小塊，
即使整個搬動的時程還是無法估計，但眼前的小塊卻是相對容易估計的。
所以如果在搬動的過程當中有緊急的事情發生，這個人可以把眼前的小塊立刻 release 出去之後立刻來幫忙。
這樣人力的調度就會變得相當有彈性。即使整個搬動的時程和工作量是拉長的，但換來人力的彈性應該是相當珍貴且划算的。


但上面的目標說得容易阿！事實上，要把一個 scope 很大的東西切成可以隨時 release 到 production 的小單元其實並不容易。
尤其在像我們這次遇到的架構的調整，由於同時線上還有 user 正在隨時使用同一個功能，
所以這些"可以隨時 release 到 production"的小單元，必須是要同時兼顧新、舊架構的。

這個部份我們有從這次的經驗中歸納出一些策略和大家分享：

### 定義出搬動的切入點

在這次搬動的過程中，我們常常遇到 "要從哪邊開始搬呢？" 的問題。眼前有好多 table, API, 要從哪一塊開始下手才對呢？
這時候我們發現有一些切入點可以用來分析：

- Data: 通常整塊大的功能會牽涉到很多存在 database 裡面的 data。而這些 data 之前是有階層/從屬關係的。好比說一個 `post` *屬於* 一個 `user` 並且*有很多* `comments` 等等。
- API: 圍繞著 data 的則是相對應的 API 和商業邏輯。例如說我們有 `post` 的 creation API, updating API 等等。
- Side Effect: 再者，API 被呼叫了之後，可能會產生除了 data 之外的 side effect。例如說有人在一篇 `post` 下面留了新的 `comment`, 這時候要寄信通知 post 的作者就是一個 side effect。

上面提到的三個切入點，是可以分開被當成 "可以 release 到 production" 的小區塊的。

以我們的例子來說，我們要搬動的功能的 data 大約牽涉到五到十個 table，而這些 table 彼此間有從屬關係。
我們的作法是：

- 找到在階層最高的那個 table
- 然後從它的 creation API 開始搬動
- 並且在新的 creation API 裡面，去保留舊的 creation API 的行為，藉此讓我們先不用搬動它相對應的 side effect 就可以把新的 creation API release 出去，也讓其他還沒搬動的 API 可以繼續正常運作。


當然，上面只是整個搬動的第一步（上面說的大約花了兩個小時就上 production 惹 XD），後面還有千千萬萬個步驟。但是透過這樣的切割，我們可以立刻 relase 搬動好的東西；也因為有東西已經 release 出去了，要做後續的估計也會相對的比較準確。

透過上面的切入點，我們在切割 "可以 release 到 production" 的小區塊的時候，可以問自己說：

- 這次我們要搬動什麼 data？
- 要從這個 data 的哪一個 API 開始搬？
- 這個 API 有沒有對應的 side effect，要在這次一起搬動嗎？

當然，在回答上面的問題的時候會有各種要考量的因素，包括本來 code 可以被重覆使用的程度，或者是 API 的複雜度等等。


### 決定 deploy 的順序

在做好 "可以 release 到 production 的小區塊"，然後也測完之後，接下來的任務就是…把它 release 到 production XD
但是在隨時有人在使用這些要搬家的功能的狀況下，要怎麼做到無痛的轉換並不是一件容易的事情。
就好像我們要幫人整修房子，但是這間房子的主人正住在裡面。
要怎麼在他沒有感覺的狀況下把房子整修好，是一件要想一下的事情。

下面是要 release 的時候，可能會遇到的切入點們：

- Creation API
- Reading API
- Client Creation Endpoint
- Client Reading Endpoint
- Data Migration

上面這些點有些可以一起被 release 出去，像是我們可以在同一個 deploy 同時換新 Creation API 和 Reading API。
但像是 API 和 Client Endpoint 則是很難確保可以在同一瞬間都同時換成新的。
另外，如果有牽涉到 schema 的轉換的話，那要把 data 從舊的 table 寫到新的 table 大部份是需要一些時間的。

綜合上面的特性，再配合 "這個 feature 我們可以容忍怎樣的不一致的行為"，就可以決定 deploy 的順序了。

下面是我們這次遇到的情境們給大家參考：

### Client Reading v.s. Client Creation v.s. Data Migration

最常見的狀況就是我們要搬動一些 data 存放的位置或方式,
例如說：我們要搬動 "建立一個 `post`" 這個行為。

這時候改變的行為有：

- 在 server side，有新的 Creation/Reading API (`[POST/GET] /posts'` => `[POST/GET] /new-posts`)
- 在 client side, 要呼叫新的 Creation/Reading API
- 要把舊的 data migrate 到新的 table (把 data 從 `posts` table 搬到 `new_posts` table)

在這樣的狀況下，如果我們先做 data migration 再搬動 client creation 的 endpoint，
那麼我們就可能會遇到 "少 migrate 一些 data" 的問題。
原因是因為，如果在我們 migrate 完 data 之後、搬動 client creation 之前的這段時間之內，有人又寫入了新的 data，
那麼這筆新的紀錄就不會被搬動到。

**所以在大部份的狀況下，我們會先搬 client creation endpoint，再來 migrate data。**

但是這樣一來，我們就會遇到下一個問題：*Client Reading Endpoint 要什麼時候搬？*

### 複雜的 Reading Operation v.s. 不一致(Inconsistent) 的 Reading Operation

在上面的情況下，"搬動 Reading Endpoint" 有兩個選擇：

- 在 migrate data 之前
- 在 migrate data 之後

如果我們在 *migrate data 之前*，就先讓 client 去讀新版的 data 的話，那這時候：

- 當 client 建立了一筆新的 record, 他可以看到正確的結果
- 但是*舊的 data client 有可能會讀不到*（因為正在被 migrate)

反之，如果我們在 *migrate data 之後*再讓 client 去讀新版的 data 的話，那這時候：

- 當 client 建立了一筆新的 record, *他會看不到結果*（因為這時候 client 還是去讀舊的 data , 但新建立的 data 是存在新的地方）
- 舊的 data 可以讀到

又如果，我們完全不想犧牲任何的讀取的一致性的話，那我們在 server side 就要建立更複雜的 reading API, *同時去讀新的和舊的 data, 然後再把結果 merge 起來回傳給 client*。

在不同的狀況之下，我們會要做不同的取捨。一些可能的判斷標準有像是這個資料本身的性質，data migration 預期花費的時間，和複雜的 reading API 的難度等等。

假設今天這個 data 是按讚的紀錄，那可能可以用第二種作法。因為建立了新資掉之後，可以在 client 用 UI 先讓使用者暫時不會發現不一致的讀取。
如果 data migration 的時間很短，那可能可以使用第一種作法，因為不一致的時間就只有一下下。
又如果這些 data 非常的重要，像是金錢的交易紀錄等等，這時候我們可能不能接受有任何 data 不一致的狀況，所以勢必要建立比較複雜的讀取 API 同時兼容新、舊的資料。

**簡而言之，要認真的考量資料的性質和可以容忍的限度，然後做出最適合的決定。**



### Robust Migration v.s. Creation Availability

在上面的情境裡，有一個步驟是要把舊的 data migrate 到新的地方去。
在某一些時候，這個行為並不是那麼的單純。**因為在 migrate data 的同時，新的 data 也正在被寫入。**
所以如果資料本身有某些限制的話(像是 unique constraint), 就有可能會遇到 migrate 失敗的問題。

舉例來說，假設我們今天要 migrate 的是使用者對文章按讚的資料。而在這個 table 裡面我們有建立 `(user_id, post_id)` 的 unique constraint。
也就是說，同一個 user 是不能對同一個文章按讚兩次的。

這時候如果我們舊有的 data 有十萬筆，其中第八萬筆是 `(user_id = 1, post_id = 1)` 這樣的 data。
而當我們 migrate 到第五萬筆的時候，又有一筆 `(user_id = 1, post_id = 1)` 被寫入到新的 table。
在這個當下，因為舊的 data 的第八萬筆還沒有被寫入新的 table, 所以這個新的寫入是合法的。
但是當我們 migrate 到第八萬筆的時候，我們就會遇到 unique constraint 的衝突了。

這個狀況只是其中的一個例子，隨著不同的資料類型，可能會遇到各種 migration 和新寫入的衝突。
要解決這樣的問題，其實就是要建立一個比較聰明的 migration，讓這個操作可以應付同一時間的寫入。
而技術上要做到這件事的難度，則會和 data 的存放方式、現有的 code quality、甚至是時程的規畫等等各種因素有關。

如果面對到這類的問題，而考量的結果是不要建立聰明的 migration 的話，那就必須要犧牲一段時間的 creation availability。
意思是說，我們就在 migrate data 的這段時間，暫時不讓使用者寫入新的 data。這樣就可以避免掉剛才的問題了。

而就像任何的技術選擇一樣，要考慮各種不同的因素。
以我們遇到的狀況，像是有一類型類似按讚的 data，我們就是這樣來 migrate 的。
因為簡單版本的 migration 很容易實作，並且跑起來很快。
讓使用者在一小段時間（以我們的狀況大約是十秒）先不能按讚一下，是可以接受的狀況。


### 如果 data 的覆蓋範圍又深又廣，可以考慮在搬家的過程中，讓新舊 data 並存一下

這個點是我們這次的搬動過程中，我覺得最有價值的一點。雖然其實道理很單純，但是有了它，才可以讓這個漫長的過程有個開始。

通常我們的 data 會有階層的關係。而最上層的那些 data，常常很多東西都跟他有關。
例如說我們有 `user`, `post`, `comment`, `like` 這些 data，通常所有東西都和 `user`(最上層的) 有關。
所以如果我們要把 `user` 搬家的時候 (假設要搬到一個新的 table 叫 `customer` 好了), 如果直接把 `user` 移除掉，那其他的 data 就很難再運作了。
但是如果我們又不想要一次把 `post`, `comment`, `like` 全部搬好再 release 的話該怎麼辦呢？

這時候我們可以在建立一個 `customer` 的時候，同時也建立一個對應的 `user`。這樣一來，我們就可以在搬動其他 data 之前，先 release "`user` 到 `customer`" 的搬動。甚至還可以延後其他 side effect 搬動的時間。
這樣一來，在分配小塊的 release 的時候會增加很多彈性。


## 後記

雖然說好像有企圖整理出一些東西，但其實不管怎樣，搬家就是很辛苦 der。
每一個步驟都會遇到不同的狀況。**但我覺得在各種方法和策略之上，更重要的是要衡量各種的狀況做出最適合的取捨。**
像是複雜度、功能不完整的時間 / 容忍度、時程等等。
畢竟精準地分析現況做出取捨正是攻城獅的本質吧！
