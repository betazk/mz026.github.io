---
layout: post
title: Hello Microservice Event Stream
comments: true
description: "當我們有一個以上的 service 要溝通的時候，我們有很多不同的作法可以完這這件事情。這篇文章會介紹其中一種作法叫作 Event Stream，以及我們要實作它的時候，尋找作法的過程和考量。"
tags:
  - rabbiqmq
  - microservice
  - "event-stream"
  - rails
---

前陣子在做[公司的新產品](https://hire.codementor.io)，慢慢的要把一部份的功能抽象出來做成獨立的 service，
一方面是讓獨立的功能可以有自己的 release lifecycle，一方面則是因為 project 小小的總是可以做比較快，沒有 legacy 的包伏阿ㄎㄎ。

Microservice 是一個很深遠的課題，但不是這篇文章要討論的主題 XD。

這篇文章的主題是：

當我們有一個以上的 service 要溝通的時候，我們有很多不同的作法可以完這這件事情。這篇文章會介紹其中一種作法叫作 [Event Stream](https://www.thoughtworks.com/insights/blog/scaling-microservices-event-stream)，以及我們要實作它的時候，尋找作法的過程和考量。


## Service 之間要怎麼講話呢？

當我們的服務分成一個以上的 service 的時候，免不了要讓它們互相溝通。

首先，什麼叫作 "分成一個以上的 service 呢"？其中最單純的 case 就是想像它們是兩個獨立運作的 server。
好比我們有一個 server 是專門管理 user data 的，像是註冊阿，修改 profile 等等。
而另一個 server 則是專門負責寄各種 email, 包括和 mailing service 溝通，處理 unsubscribe 等等。

在一些情境之下，這兩個 service 必需要相互溝通。
例如說： 我們有一個新的 user 向 user service 註冊了一個新的帳號，這時候我們需要寄一封認証信給這個 user,
於是 user service 就會和 mailing service 說："yo! 我這邊有一個新註冊的 user，他的 email 是這樣，請你幫我寄信認証信給他。"


像這樣就是其中一種 service 之間需要溝通的情境。

而 service 之間的溝通，可以分為兩類：

- syncrhonous 的溝通
- asynchronous 的溝通 (~~廢話XD~~)


### Synchronous 的溝通

Synchronous 的溝通指的是 consumer 會在一個動作之內完成和另一個 service 的溝通，像是下面的 psudo code：

```ruby
user = User.create(...)

result = MailingService.send_confirmation_mail(user)

if result.success?
  #...
else
  #...
end

```

### Asynchronous 的溝通

Asynchronous 的溝通則是指說，**對於呼叫的一方，不會立刻得到回應**。而其中一種作法叫作 [Event Stream](https://www.thoughtworks.com/insights/blog/scaling-microservices-event-stream), 也是這篇文章要討論的主題。

Event Stream 指的是 consumer 要和另一個 service 溝通的時候，是用**放出一個事件** 的方法來達成。
以寄認証信的例子來說，當我們有一個新的 user 註冊了，**user service 不是直接請 mailing service 寄信，而是放出一個事件說 "有人註冊惹！"，然後 mailing service 在接到這個事件之後，再去做對應的處理（寄信給user)**

如果用 pseudo code 來表示就像：

```ruby
user = User.create(...)

EventPublisher.publish({
  event_name: 'user-registered',
  payload: {
    email: 'the-user@test.com'
  }
})
```

這樣的作法，在概念上很像 javascript 的 event，或是 design pattern 裡的 [observer pattern](https://en.wikipedia.org/wiki/Observer_pattern)。
在概念上我們可以發現，後者的做法裡面，**我們 decouple 了 user service 和 mailing service。
也就是說，對於 user service 來說，它是不知道 mailing service 的存在的。**


### 哪一種比較好勒？

其實這兩種 approach, 並沒有顯的好壞，而是各有優缺點。在不同的情境下會有各自適合的選擇（就跟大部份的程式設計選擇一樣 XD）

以第一種作法來說，它的優點是從 consumer service 我們就可以知道整串行為下來會發生什麼事情。實作起來也相對單純一些。
但最明顯的缺點之一，則是我們把不同的 service couple 起來了，也就是說，如果我們修改了mailing service 的 API （好比說改了 api endpoing），這時候 user service 也必需做對應的修改。這種情形當 service 數量一多的時候是很可怕的。

而第二種作法的優點主要就是我們 decouple 了各個 service，概念上像是讓每個service自主管理，這也是把功能拆成不同 service 的主要目的之一。但缺點則是"一個動作會連帶哪些行為" 這個問題變得比較難回答了（甚至無法回答）。

另外，每個 service 執行的時候，有可能會在半路因為各種原因失敗。這時候要做 error handling 的時候，在 syncrhonous 和 asynchronous 的做法之下，也會遇到不同的挑戰。


大致上來說，如果不同 service 之間的行為是互有相依性的，那多半只能選擇 syncrhonous 的做法。
反之則可以考慮 asynchronous。但話說回來，"不同 service 之間的行為是互有相依性的" 這個狀況，可能也和 service 的切割有關。
也就是說，是有可能透過不同的 service 界線定義，讓本來有相依性的狀況變成沒有。
但"如何定義 service 的邊界"這個問題是相當深難的，無法在這個文章裡面仔細討論 XD

總之以我們的 case 來說，我們需要一個 asynchronous 的 solution！！

## Requirements & How it works

以 service 之間的 event stream 來說，我們的需求和預期的運作方式大致上是這樣：

- 當有一個事件在某個 service 上發生的時候（好比說註冊），這個 service 就放出一個事件
- 這時候會有一個或以上的 service 同時 subscribe 這個事件，這些 service 彼此之間是互相獨立的，也就是說，它們可以是不知道彼此之間的存在的
- 對於每一個有 subscribe 事件的 service 來說，它們各自控制自己的事件處理 cycle，包括有 error handling 等等

這個**負責統整各式事件的東西有一個名字叫[message broker](https://en.wikipedia.org/wiki/Message_broker)**, 概念上來說，各個 service 互相其實不管彼此，而是統一透過 message broker 來傳遞事件。

簡單地說，我們需要一個 message borker 來幫我們做 [publish/subscribe](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) 這件事情。

## 可以用 Background Job 解決嗎？

在討論過程中，有考慮過

"咦，那可以用像是 [delayed_job](https://github.com/collectiveidea/delayed_job) 或是 [sidekiq](https://github.com/mperham/sidekiq) 這類的 background job 來解決嗎？"

結論是：

"要的話也是可以，但是蠻麻煩的"

怎麼說呢？
主要的差別在於，對於 event stream 來說, subscribe 同一個事件的 service 通常有一個以上。而一般background job處理的邏輯則是"一個事件(job)處理掉就把job 從 queue 裡面刪掉了"。也就是說，如果真的要用 background job 來作為 message broker 的話，那每個 subscriber 要有一個自己的 job queue來收自己有 subscribe 的事件。

以上面的例子來說，如果我們在註冊事件發生之後，email service 和 payment service 同時要做相對應的處理，那麼這兩個 service 都要 maintiain 一個自己的 job queue，並且要由一個中央的 service 負責去通知他們說 "嘿！有你subscribe的事件來囉！"



## 中意的候選人們

接下來就是要找尋合適的技術選項來完成上面的需求惹。
在這方面，我們的需求方向有一些期望：

- event stream 在 service 數量多起來的時候，它的 load 是有可能會很可觀的。所以如果可以的話，找一個穩定一點的 [SAAS](https://en.wikipedia.org/wiki/Software_as_a_service)/[IAAS](http://searchcloudcomputing.techtarget.com/definition/Infrastructure-as-a-Service-IaaS) 來做，而不要自己架硬幹
- message broker 要做得好有蠻多細節要handle，所以如果可以的話，找一個穩定一點的 library，不要自己做硬幹

1. AWS [SNS](https://aws.amazon.com/sns/) + [SQS](https://aws.amazon.com/sqs/)

從第一點需求出發，直接先看AWS有沒有簡單可以直接用的service，但找了一下發現目前最接近的選項是 [SNS + SQS](https://www.infoq.com/articles/AmazonPubSub)，可惜這樣的作法對我們來說還是太複雜了。基本上這樣的架構其實和上面提到的用 background job 直接做差不多。

2. [RabbitMQ](https://www.rabbitmq.com/)

另外一個選項就是 [RabbitMQ](https://www.rabbitmq.com/)。以功能來說，RabbitMQ 算是完全 cover 了我們的需要。所以最後就決定是它了！


## Hello RabbitMQ

RabbitMQ 是一個功能複雜又強大的 message broker。底下其實是 follow 了一個叫作 [AMQP](https://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol) 的協定，實作上是用 Erlang 寫的。 ~~但其實我們並不在乎這些~~ ，以使用上來說，個人認為它的資源和社群都很不錯，感覺就算踩中雷旁邊也還會有別人。以我們的狀況來說算是蠻適合的。

但下一步就是要怎麼 host 這個東西的問題。根據上面的考量，基本上最好是可以~~付錢解決掉~~找到一個SAAS最好。又因為我們的 server 是架在 heroku 上面，所以直接先從有 Addon 的開始，於是就找到了 [CloudAMQP](https://www.cloudamqp.com/)。目前使用起來都還算可以。

關於 RabbitMQ 在 application layer 的使用上後來有陸續遇到一些問題。但那就等之後再整理成文章好惹。

## 後記

在面對架構性的東西要選擇不同的 solution 其實會有各種不同的考量，很難界定說某一個選項就是比另一個好。
在這種時候，像是 team 的人力、時間、預算、流量或產品的性質都會是考量的參數。
我們目前在選擇的時候，社群的支援度還是一個很大的考量點。

而不管最後選用了什麼選項，在 application layer 方面，如果可以的話盡量把程式設計成不要讓這些選擇太緊密的和 application couple 在一起應該會是不錯的作法。
