---
layout: post
title: 大戰 Rails Connection Leak
comments: true
description: "前陣子我們的 database 遇到了疑似 connection leak 的問題。在經過一番苦戰之後，總算讓 server 恢復了平靜。
這篇文章會紀錄下整個故事，希望讓看到的人不用再踩同樣的雷一次"
tags:
  - rails
  - "connection-leak"
  - database
---

前陣子我們的 database 遇到了疑似 connection leak 的問題。在經過一番苦戰之後，總算讓 server 恢復了平靜。
這篇文章會紀錄下整個故事，希望讓看到的人不用再踩同樣的雷一次ㄎㄎ。


## 前言


不知道從什麼時候開始，我們的 database 開始出現了 connection count 爆破的問題。
基本上就是 exception 噴出來說連不進去 database 惹，連要用 `psql` 直接連進去也無法。(阿就 connection 滿了咩)
這時候的解法基本上就是把 server 重開一發。 ~~然後就可以當作什麼事都沒發生一樣去睡覺~~
一開始的時候這種事情的頻率並不高，大約兩、三個禮拜一次，並且我們直覺這種問題應該是蠻麻煩搞的。
所以就這樣放著。
但一直到最近，它發生的頻率越來越高，工程師的羞恥心再也不容許我們坐視不管。

### 背景介紹

我們的 database 是 host 在 RDS 上面的 postgresql，然後 application layer 是跑在 Heroku 上面的 rails。
基本上算是一個常見的 stack，要重開也就是一個 heroku command　就搞定。
在 Heroku 上面除了有 serve API 的 server 之外，還有 background job 若干、RabbitMQ worker 若干。

## 症狀診斷

首先，我們先在 RDS 上面加上 [cloud watch](http://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/rds-metricscollected.html) 的 alert，讓它在 connection count 快要爆破的時候可以先通知我們。
這樣一來，即使無法正面解決問題，但至少可以讓我們在快要爆之前，可以連進去 database 看到底是發生了什麼鬼事。

於是，在某幾次接到 alert 的時候，我們立馬連進去看看到底是什麼東西在搞鬼。
這邊我們用一個叫 `pg_stat_activity` 的東西來看。

如果我們下：

```
\d pg_stat_activity
```

會得到下面的欄位：

```
+------------------+--------------------------+-------------+
| Column           | Type                     | Modifiers   |
|------------------+--------------------------+-------------|
| datid            | oid                      |             |
| datname          | name                     |             |
| pid              | integer                  |             |
| usesysid         | oid                      |             |
| usename          | name                     |             |
| application_name | text                     |             |
| client_addr      | inet                     |             |
| client_hostname  | text                     |             |
| client_port      | integer                  |             |
| backend_start    | timestamp with time zone |             |
| xact_start       | timestamp with time zone |             |
| query_start      | timestamp with time zone |             |
| state_change     | timestamp with time zone |             |
| waiting          | boolean                  |             |
| state            | text                     |             |
| backend_xid      | xid                      |             |
| backend_xmin     | xid                      |             |
| query            | text                     |             |
+------------------+--------------------------+-------------+
```

在這個情境下，我們需要的主要是：

- query_start: query 從什麼時候開始的，如果一個 query 老早就開始了(理論上也是老早就結束)，但到現在卻還連在那邊，那一定不單純阿。
- query: query 的內容，看正在執行的 query 是什麼東西，可能可以反推出是哪邊的 code 出問題。
- state: query 的狀態，大部份的時候應該是 idle

(詳細的欄位說明可以參考 [postgres doc](https://www.postgresql.org/docs/9.5/static/monitoring-stats.html#PG-STAT-ACTIVITY-VIEW))

在下了 query 之後：

```
SELECT query_start, query, state FROM pg_stat_activity WHERE pg_stat_activity.datname = 'my-database-name' and state = 'idle' order by query_start;
```

確實有發現一些 conneciton idle 了很久，但是 `query` 的內容卻都很發散。也就是說，在這個階段並沒有得到什麼有用的資訊 QQ。


## 開始 survey

既然第一眼無法看出個所以然，看來必須要做比較詳細的 survey 了。在一陣 google 之後，找到了跟我們症狀很類似的東西： **Connection Leak**

## Hello Connection Leak

在了解 conneciton leak 之前，首先要先知道 Rails[(note1)](#note1) 是怎麼控制 database connection 的。
簡單來說，Rails 會 maintain 一個 [connection pool](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/ConnectionPool.html)，概念上大致上是說，Rails 會自己拿著一些 database connection，

- 當 application 需要連 database 的時候，就會從這個 pool 裡面拿出(check out) 一個 connection 來使用
- 當 application 用完 connection 之後(例如說當 API request 結束, respond 給 client 了)，就必須把這個 connection 還給(check in) 這個 connection pool，以便後續的 request 可以繼續使用。

**而所謂的 connection leak， 就是如果我們做了第一步(把 connection 拿出來)之後，卻沒有做第二步(沒有把 connection 還給 connection pool)。這時候我們可以想像 connection pool 裡面的 connection 就會越來越少，~~然後他就死掉了~~然後我們就完蛋惹。([note2](#note2))**

以上這兩個步驟，基本上在 "大多數" 的使用情境下，developer 是不會察覺的。也就是說，大部份的時候我們在寫 Rails 的時候，並不會特別去 check out/in database connection。為什麼呢？因為 access database 這個境境太常見了，幾乎每一個 request 都會做這樣的事情。所以 rails 自動幫我們處理掉 check in/out connection 的行為。

**但上面的"大多數"，指的是 "在沒有手動開新的 thread的前題下"**。
也就是說，如果在 request 裡面另外開 thread，並且這個 thread 有去連 database，那麼這個時候，rails 會幫我們做第一步(check out)，但卻不會幫我們做第二步(check in)。所以如果我們在 request 裡面有另外開新的 thread，就要特別注意要自己處理 check in 的行為。

處理的方法大致上是用 connection_pool 的 [`checkin` method](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/ConnectionPool.html#method-i-checkin)

```ruby
ActiveRecord::Base.connection_pool.checkin(connection)`
```

或者是用 connection_pool 的 [`with_connection` method](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/ConnectionPool.html#method-i-with_connection)

```ruby
ActiveRecord::Base.connection_pool.with_connection(&block)
```

**但不管是用哪一個，都超麻煩！**因為~~誰會記得阿靠！~~如果另開 thread 的情況很常見的話，是很難確保每個地方都有照顧到這樣的行為的。再者，如果真的發生的話，要在茫茫 code 海裡面找到是哪個地方忘記 check in connection 簡直是太痛苦阿！


在 survey 到疑似兇手的 connection leak 之後，因為我們的 code 裡面確實有使用一個 library 會時不時會開 thread 去處理一些東西。所以我們高度懷疑就是它。但實際試了之後卻無法 reproduce，在仔細檢查之後，也發現其實在使用這個 library 的地方，都有做好上述的處理了。

此時案情再度陷入膠著~~ㄇㄉ~~


## [天眼通](https://i.imgur.com/ueTxNy3.jpg)

在搞了一陣之後發現找不到原因有點絕望之後，這時候突然想到說，阿靠，會不會是設定的問題呢？結果一看之下還真的是。(這不是通靈，什麼才是通靈)

## Connection Pool 的設定

[`ConnectionPool`](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/ConnectionPool.html)是用一個叫作 `pool` 的參數來控制 pool size (connection pool 最大可以拿在手上的 connection 數)，**即使 connection pool 並不會一開始就把這些 connection 全部都拿在手上，但從 database 的角度來說，我們可以把這個數字視為 "database 必須要準備好有這麼多 connection 可能會連進來"**

**但是不是 `pool` 設定成 `n` 就代表 database 最多會有 `n` 個 connection 連進來呢？答案是不一定。**

不一定的原因是因為要看我們的 application 是用 process based (像是 [unicorn](https://devcenter.heroku.com/articles/rails-unicorn)) 還是 thread based 的 server (像是 [puma](https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-server))，另外還和我們 host 的環境有關，例如說以 Heroku 來說，就會和我們跑的 dyno 個數有關。

### Process Based Server

在 Process based 的 server 下，我們會有一個 master process。
而在每一個 request 進來的時候，我們都會從 master process 新開(fork) 一個 worker process。**這時候 pool size 的 scope 是 "master process"**。
而如果我們的 request 沒有另外開 thread 的話，那麼一個 worker process(request) 只會需要一個 connection。
這時候 pool size 的設定其實就看我們的 master process 最多會 fork 出幾個 process 而定。

以 unicorn 為例，假設我們最大的 worker process count 是 3：

```
worker_processes 3
```

那在大多數的境況之下，我們的 pool size 就只要設定成 3 就夠了。
在這個時候，會同時連到 database 的 connection count 最多就會是 3。

但如果我們用同樣的設定，在 heroku 開了 5 個 dyno 的話，
那同樣的設定就會被複製5倍，也就是同時會有 15 個 connection 同時連到 database。

### Thread Based Server

在 Thread based server 下，我們會設定若干個 worker process。當一個新的 request 進來的時候，會由 worker process 開(spawn)一個新的 thread 來處理它。
**這時候 pool size 的 scope 是 "一個 worker process"**

以 puma 為例，
如果我們的 worker process 是 2，每個 worker 可以開 5 個 thread：

```ruby
# config/puma.rb

workers 2
threads 5, 5
```

這時候在大多數的狀況下，我們的 pool size 設定成 5 就夠了。
而同時會連到 database 的 connection count 則是 10。

同上，如果我們用這樣的設定在 heroku 上面開了 4 個 dyno 的話，
那同時會連到 database 的 connection count 最多就會是 40。


### 結果：

結果後來發現有點搞笑，我們遇到的問題其實就是 pool size 設太大，
導致 Rails 會企圖拿超多 connection，
於是當 through put 大的時候，connection 的量就直接打到 database 的上限了。



## 後記

### 1. 少碰 thread，請愛用 background job

這個有點個人偏好其實。但我真心的認為 thread 實在是很難掌握。這種事就是在你越不想，越沒空處理他的時候，他就會跑出來咬你一口。
所以如果可以的話，大部份的事情應該是可以用 backgroud job 達成的。像是 [sidekiq](https://github.com/mperham/sidekiq), [Resque](https://github.com/resque/resque), [delayed_job](https://github.com/collectiveidea/delayed_job) 都是成熟且社群完整的選擇。


### 2. 設定的調整是無止盡的

上述的設定調整其實只 cover 了一小部份。實際上要調整各種參數，要考量的東西非常多，並且現在很好的設定，到了幾個月後 application loading 改變之後，也未必適合。
簡單的說就是要常常關心一下 performance 或者是讓 performance 來關心你 (設定一些 alert 之類的)，
常常看看它們，就可以跟它們當好朋友。

### 3. 不要問我怎麼通靈 debug, 我不會。


#### note1：

精確地說，應該是 ActiveRecord 在控制 connection

#### note2：

後來覺得其實這種狀況，和我們遇到的狀況有點不同。這邊提到的 connection leak 的狀況應該是 Rails connection pool 的 connection 用完了，所以應該是會看到

```ruby
ActiveRecord::ConnectionTimeoutError
```

這樣的錯誤，而不是整個 database 連不進去才對。

#### References:

- [ActiveRecord Concurrency in Rails4: Avoid leaked connections!](https://bibwild.wordpress.com/2014/07/17/activerecord-concurrency-in-rails4-avoid-leaked-connections/)
- [Concurrency and Database Connections in Ruby with ActiveRecord](https://devcenter.heroku.com/articles/concurrency-and-database-connections)
