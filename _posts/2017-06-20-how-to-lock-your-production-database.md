---
layout: post
title: 第一次 lock production database 就上手
comments: true
description: "前陣子在 migrate database 的時候，發生了慘烈的 production database lock 災情。 在救火的過程當中雖然痛苦，但是也因此學到了很多以前不知道的東西。 在這篇文章裡面，我還是會把一些過程、策略和方法紀錄下來。"
tags:
  - database
  - postgres
  - lock
  - migration
---

前陣子在 migrate database 的時候，發生了慘烈的 production database lock 災情。
在救火的過程當中雖然痛苦，但是也因此學到了很多以前不知道的東西。
我想對於更有經驗的 developer 來說，這邊很多東西應該都是常識。
但在這篇文章裡面，我還是會把一些過程、策略和方法紀錄下來。

## 情境介紹

在 Codementor, 我們的 database 用的是 postgres，架在 RDS 上面。
server 的 host 則是 heroku。

當時我們打算在 users table 上加上一個

```sql
alter table users add column xxx bool not null default false
```

基本上有經驗的 developer 看到這邊，就知道後來會發生什麼事情了。

{: .center}
![](https://media.giphy.com/media/pAnrp75JZ8JDG/giphy.gif)

即使隱約知道在大的 table 上跑這樣的 migration 應該要多做一些什麼才對，
但是當時的我們內心的小惡魔告訴我們：

*網路上說的那些，是要有很大 scale 的 data 才會遇到的問題啦。先推推看嘛，推嘛推嘛～*

於是開始有人回報說整個服務掛了，New Relic 開始寄來各個地區 ping 不到 server 的信, 焦慮指數滿點。

接著過了大約十五分鐘，整個服務總算復活惹，阿彌陀佛。

## 賽後檢討

但其實我們在推到 production 之前，是有上 staging 試過的。
我們的 staging 的環境在當時，有和 production 幾乎一樣的 data 量 (RDS restore)，
stack 的話也和 production 一樣，是 heroku + RDS。
而機器的 scale 和 production 相比是比較弱的。

當時我們在 staging 上測試的結果，整個 migration 大約跑了一分鐘左右。
所以我們判斷，既然 production 的機器比較強力，應該小於一分鐘就可以完成了吧。

所以對於我們來說，心中的兩個疑點是：

- 在 migration 的時候到底發生了什麼事？為什麼會讓整個服務死掉呢？
- 為什麼在 staging 的環境下可以順利跑完，但在 production 的環境下卻死了十五分鐘呢？


## 關於 migration

在google上面不難找到關於 [migration 要注意的事項](https://www.braintreepayments.com/blog/safe-operations-for-high-volume-postgresql/)。但關於像我們這樣的 migration 如果不小心跑下去了會發生什麼事情，卻比較少有相關的資源。經過一輪 google 之後總算發現[有人跟我們遇到類似的狀況](https://gocardless.com/blog/zero-downtime-postgres-migrations-the-hard-parts)。於是我們總算知道發生了什麼事情：

### 1. 關於 lock
事實上，在 database 的各種操作下，常常會對於不同的 row 或者是 table 產生各種[等級不一的 lock](https://www.postgresql.org/docs/9.4/static/explicit-locking.html)。
隨著 lock 的等級不同，當某些互相衝突的 lock 企圖佔住同一個資源(像是 table 或者 row)的時候，那麼後面的操作就必須要等到前面的操作先做完了(把衝突的 lock 釋放掉)才可以開始進行。

### 2. alter table

就像上面提到的，不同的 database 操作會產生各種不同等級的 lock。而很不幸的，大多數的 `alter table` migration 會產生的 lock 會是最高等級的一種lock: `Access Exclusive Lock`。

> ALTER TABLE changes the definition of an existing table. There are several subforms described below. Note that the lock level required may differ for each subform. An ACCESS EXCLUSIVE lock is held unless explicitly noted.

所謂"最高等級"的lock，可以把它想像成是最難相處的一種 lock，也就是說，它和所有其他等級的 lock 都是互相衝突的。所以當這種 lock 出現的時候，它是不能和其他等級的 lock 並行的。

### 3. 所以 migration 的當下究竟發生什麼事呢？

當我們進行 alter table 的時候，postgres 會在這個過程當中，企圖用 `Access Exclusive Lock` 鎖住：

- alter table 的 對象本身：以我們的例子來說，是 `users` table
- **用 foreign key reference 到這個 table 的其他 table 們**：這個就有點羞赧了。因為以我們的 database 來說，根本就有超多 table reference 到 `users` table 阿阿阿，難怪那麼可怕 QQ

但就像上面提到的，當不同等級的 lock 互相衝突的時候，後來的要等前面的。也就是說，當我們要對 `users` table 進行 `alter table` 的 migration 的時候：

- 如果當時我們企圖 lock 的 table(以我們的case來說是有超多 QQ)，如果有其他的 table 正在被 lock 住，那這個 migration 要等到這些其他的 lock 都被釋放掉了才能開始
- 並且從這個時候開始，**所有在這個 migration 開始的時間點後面發生的 lock 全部都要等到這個 migration 結束之後才能開始。**而 `Access Exclusive` 是等級最高的 lock，所以在這個 migration 開始的瞬間起，`users` table 和其他 reference 到 `users` 的 table 們是完全不能使用的。


這就解釋了我們的第一個疑問：在 migration 的當下究竟發生了什麼事。


## 那為什麼 Staging 沒事呢？

但是上面的一切，並不能解釋 staging 和 production 的差距。

### 得到 lock 的時間

我們立刻想到，production 和 staging 還有一個差別是 traffic。*有沒有可能因為 production 上面的 traffic 很多，所以在 "migration 排隊得到 lock" 的時間可能相當的長，因此造成了 production 和 staging 的差距呢？*

這時候最好的方法就是在 production 上面實驗看看惹 XD

當然，我們無法再跑另一個類似的 migration。但如果只是要確定"得到 users table lock 的時間"，我們可以用下面的方法：

```sql
BEGIN;
LOCK TABLE toys IN ACCESS EXCLUSIVE MODE;
COMMIT;
```

在上面的指令，我們把 `users` table 用 `Access Exclusive` lock 鎖住，然後立刻放掉。
用這樣的方式，就可以摸擬出 "得到 users table lock" 的時間。
但在 production 上 traffic 差不多的狀態下試的結果，發現只要大約一、兩秒就完成了。

也就是說，"得到 users table lock" 的時間，並不足以解釋 production 和 staging 上面 15 分鐘和 1 分鐘的差距。

當然，實際上在 migration 的時候，我們 lock 的不只有 `users` 這個 table，而是有千千萬萬個其他 reference 到 `users` 的 table 們。
但基於 users table 得到 lock 時間的 scale 和 15分鐘差太多，再者，從log 上看起來，在 migration 之前，並沒有明顯的 slow query記錄，所以我們初步排除了 "得到 lock 的時間"的這個因素。

### 直接複製一個環境打打看

後來我們嘗試了複製和 production 一樣的環境，然後再跑一次 migration 看看。
首先，我們大致上計算了當時的 throughput 和 request 的內容，
並且把 application server(Heroku dyno) 和 RDS 都開到和 production 一樣的 scale，並且連上一樣的 connection count。

在企圖重現當時的狀況之後重跑一次相同的 migration，

發現還是無法重現 15 分鐘的狀況阿阿阿！！！


基本上，關於 "為什麼那個時候的 production 和 staging 差這麼多" 這個問題到現在還沒有解答。如果有人有可能的答案請提點我一下，霸拖霸拖！


## 心得與感想

migration 要小心，不要不信邪。網路上說故事的都是真的 XD。
下面是一些資源可以參考：

- [Safe Operations For High Volume PostgreSQL](https://www.braintreepayments.com/blog/safe-operations-for-high-volume-postgresql/)
- [Zero-downtime Postgres migrations - the hard parts](https://gocardless.com/blog/zero-downtime-postgres-migrations-the-hard-parts)
- [ALTER TABLE and downtime, part II](http://www.databasesoup.com/2013/11/alter-table-and-downtime-part-ii.html)
- [Explicit Locking (postgres doc)](https://www.postgresql.org/docs/9.4/static/explicit-locking.html)
- [Exploring Query Locks in Postgres](http://big-elephants.com/2013-09/exploring-query-locks-in-postgres/)
- [Common misconceptions about locking in PostgreSQL](https://www.compose.com/articles/common-misconceptions-about-locking-in-postgresql/)

