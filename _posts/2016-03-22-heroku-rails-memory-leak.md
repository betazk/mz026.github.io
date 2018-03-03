---
layout: post
title: Heroku Rails Memory Leak
comments: true
description: "最近手上的 rails project 遇到了 memory 的問題。經過一番努力之後，有發現一些原因也做了相對應的處理。把一些東西記錄在這邊。"
tags:
  - backend
  - ruby
  - "memory-leak"
  - rails
---

最近手上的 rails project 遇到了 memory 的問題。經過一番努力之後，有發現一些原因也做了相對應的處理。把一些東西記錄在這邊。

我們的 project 是 host 在 [Heroku](https://www.heroku.com/) 上面的 Rails project。發現 memory 的問題的症狀是 Heroku 一直噴 [R14](https://devcenter.heroku.com/articles/ruby-memory-use) (memory quota exceeded)。就此開啟了 memory 狩獵之旅 XD

## 什麼是 Memory Leak?
從 [wiki 上面的定義](https://en.wikipedia.org/wiki/Memory_leak)上來說， memory leak 是指：
> In computer science, a memory leak is a type of resource leak that occurs when a computer program incorrectly manages memory allocations in such a way that memory which is no longer needed is not released.

也就是說我們不需要用的的東西，由於某些失誤，導致我們無法釋放這些 memory, 久而久之隨著 app 越用越久，app 使用的 memory 就越來越多，最後機器上面的 memory 就不夠了。

在像 ruby 這種有 garbage collection 的語言裡面，通常這種狀況是發生在有不要用的東西被 reference 到，導致 GC 無法回收 memory。好比說在一個 global 的 namespace 下面不小心一直積太多東西等等。

## Memory Leak on Server

Memory 的問題在各個環境下都有可能發生，包括在 mobile app, browser, 或 server 上面等。在 server 上的問題比較麻煩的地方就是，**你不知道要怎麼 reproduce 它**。由於 server 的運作是被動的接收 request，所以當我們發現有 leak 的時候，通常是看到某種 "時間-memory用量" 的數據之後才發現說 "阿，在某某時候 server 的 memory 爆衝了"，接著我們才去看 request log, 看看那個當下到底發生了什麼事情。但是通常 server 的 request 數量，在短時間之內就會有超極多，所以幾乎無法用肉眼直接看出說 "喔喔！就是這個 page/api 害的！"，我認為這是當 memory issue 發生在 server 上的時候的難處。

## Ruby
我們的 server app 是用 ruby on rails 寫的，其中 ruby 用的是 MRI (c ruby)。理論上 ruby 有行之有年的 garbage collection 機制，所以當我們 server 出現不尋常的 memory 使用爆衝的時候，應該是先從自己的 application 開始找才對，一方面是以機率上來說，我覺得自己 application 的失誤應該是 >> C ruby 的失誤。再者，即使最後真的是 c ruby 的失誤，那其實我們能做的大多也只有等他的 patch 而已 XD

## 我們真的有 memory leak 嗎?
在看到 server stats memory usage 步步高升之後，其實先不是急著找 memory leak。原因是因為很多時候 server 會 lazy load 很多東西，也就是說他一開起來的時候拿到的 memory 量會少於最後真正要用的。

**要判斷是不是真的有 leak，要看 memory 用量狀況有沒有隨著時間而達到一個飽和的值。**如果 memory 用量有隨著時間趨近於某一個值的話，那應該就不是 leak 才對。那當 memory 用量一直衝高的時候，要怎麼知道他會不會達到飽和呢？(就是要怎麼確定它是不會飽和，還是只是還沒飽和呢？)我的做法就是先把機器 memory 再開大一點，大到你覺得 "靠！都衝到這麼高了應該是沒救了吧！" 為止。

## OjectSpace dump
在確定 server 真的有 memory leak 之後，接下來就是要看到底是什麼地方出了問題。在經歷了一番 google 之後，發現 ruby 有一個叫 [`ObjectSpace`](http://ruby-doc.org/core-2.2.0/ObjectSpace.html) 的東西，可以幫我們 dump 出在某一個特定時間之內的所有 objects, 包括它們的 size, file, address 等等。

OK！那我們 dump 出這些東西之後下一步是什麼呢？在正常的狀況下，dump 出來的東西會有成千上萬個，(在server上可能會有大約一兩百萬個左右) 也就是說用肉眼直接看是看不出什麼鬼的。但我們可以用比較聰明的方法來做(註1)：

假設我們把任何一個時候，程式裡面所有的 object 分成三類：

- 維持程式正常運作所需要的那些
- 在不久的將來會被 garbage collector 回收掉的那些
- leak 的那些

很明顯的，我們真正在乎的是第三種：leak 的那些。
如果我們在程式運行當下，選擇三個不同的時間分別去 dump 那個當下的所有 objects, 我們會得到下面的圖：

<img class="center" src="http://user-image.logdown.io/user/8925/blog/8821/post/668542/pLQeU3hS6mW4RejIIYjt_3-snapshot-dump.png" alt="3-snapshot-dump.png">

其中：
- 綠色的部份就是 "維持程式正常運作所需要的那些"
- 黃色的部份是 "在不久的將來會被 garbage collector 回收掉的那些"
- 而紅色的部份是 "leak 的那些"

在得到這三個 snapshot 之後，接下來我們要做的事情就是：

**從第二個 snapshot 出發，**
- **把同時也在第一個 snapshot 裡面的東西扣掉**：如此一來我們會剩下圖中的 G2 和 L2
- **把不在第三個 snapshot 裡面的東西扣掉**：這樣一來我們就會得到 L2 了，**就是從第一個 snapshot 到第二個 snapshot 這段時間之內 leak 的 object。**

## Implementation
有了上面的概念之後，我們就可以實際上來做了。
要得到上面的 snapshot，我們要做的事情是，在程式的最一開始去 `require 'objspace'`，然後再讓它開始 track, 最後在某個時候 dump 出來：

```ruby
require 'objspace'
ObjectSpace.trace_object_allocations_start

# your application code here

GC.start
io = File.open("snapshot_file", "w")
ObjectSpace.dump_all(output: io)
io.close
```

在 rails project 的話，可以把最上面兩行放在 `config/application.rb` 之類的地方。

接下來的問題是，**如果是 rails app 的話，要怎麼把 snapshot dump 出來呢？**

如果是在 local 的話，(或你可以 ssh 進去 server 的話)，可以用 `rbtrace` 來做到這件事情：

- 在 rails app 裡面 `require 'rbtrace'`
- 然後在 command line 下這行

```
bundle exec rbtrace -p #{pid} -e 'Thread.new{GC.start;require "objspace";io=File.open("dump_file", "w"); ObjectSpace.dump_all(output: io); io.close}'
```

其中 `pid` 是 server process 的 id

**那如果是在 heroku 上面，也就是我們無法 login 進去的話怎麼辦呢？**

我之前的做法是把它寫成一個需要權限的 API，然後把 dump 出來的檔案下載下來。
(基本上在 production trace object allocation 會讓你的 app 變得很慢又很吃 memory, 所以如果可以在 local 做就儘量在 local 做，真的要在 production 上面做的話，弄完記得趕快刪掉 XD)

## Snapshot format
在經歷千辛萬苦終於到手的 snapshot 大概長這樣：

```json
{"address":"0x7f9cff6dc150", "type":"STRING", "class":"0x7f9cff7340d0", "embedded":true, "bytesize":1, "value":"w", "encoding":"UTF-8", "file":"test.rb", "line":20, "method":"dump", "generation":7, "memsize":40, "flags":{"wb_protected":true}}
{"address":"0x7f9cff6dc290", "type":"STRING", "class":"0x7f9cff7340d0", "frozen":true, "embedded":true, "fstring":true, "bytesize":13, "value":"require_paths", "encoding":"US-ASCII", "memsize":40, "flags":{"wb_protected":true, "old":true, "uncollectible":true, "marked":true}}
{"address":"0x7f9cff6dc2e0", "type":"STRING", "class":"0x7f9cff7340d0", "embedded":true, "bytesize":12, "value":"result1.dump", "encoding":"UTF-8", "file":"test.rb", "line":29, "generation":7, "memsize":40, "flags":{"wb_protected":true}}
{"address":"0x7f9cff6dc3a8", "type":"STRING", "class":"0x7f9cff7340d0", "frozen":true, "embedded":true, "fstring":true, "bytesize":9, "value":"@metadata", "encoding":"US-ASCII", "memsize":40, "flags":{"wb_protected":true, "old":true, "uncollectible":true, "marked":true}}
```

基本上就是一個有很多行很多行的檔案，一行是一個 `json` object，所以可以簡單的用各種東西去 parse, 進一步做上面提到的分析。如果是用 ruby 來做分析的話，我寫了一個簡單的版本[在這兒](https://github.com/mz026/memory-hunter)。

## 初步成果
在經過上面的分析之後，發現兇手竟然是 [NewRelic](https://discuss.newrelic.com/t/ruby-newrelic-rpm-memory-leak/27662)!! 暫時把 newrelic 移掉之後，memory 用量的狀況是改善很多。以為從此天下太平，結果並沒有阿！

## Background Job
在把 newrelic 先移掉之後，發現時不時還是有 memory 狂噴的情形。像這樣：

<img class="center" src="http://user-image.logdown.io/user/8925/blog/8821/post/668542/4BAT6Kg8RNq4RAEKPbuA_memory.png" alt="memory.png">

但不幸的是，用上面 `ObjectSpace`的做法也看不出個所以然來。這時候老衲心生一計：何不看看 [Papertrail](https://papertrailapp.com/) 的 request log?
在把 Papertrail 上面的 log archive 下載下來並且做了一些整理之後，做了以下的分析：
假設 memory 激增的時間是區間 A，為期十分鐘，

- 找到區間 A 內被 request 的 page/api
- 找到區間 A 的前十分鐘被 request 的 page/api
- 找到區間 A 的後十分鐘被 request 的 page/api

然後把上面三個拿來比較，看看區間 A 裡面究竟是比別人多做了什麼好事。
在分析之後，發現這個區間之內確實有比相鄰的區間多出某些 page 的 request, 但是重複打這些 page/api 發現也是無法 reproduce 阿！！怎麼辦？？

這樣的結果雖然令人崩潰，但是卻開啟了另一道曙光：

**如果 request 本身無法 reproduce memory 的爆衝的話，那不是代表真正的兇手在 request 以外的地方嗎？像是 backgroud job 之類的！(右手背拍左手心)**

從這個方向出發，果然發現了有一些 background job 有一些很大的 query, 會一次 load 進來很多 objects, 所以嚴格來說這些不算是 leak, 而是在短時間之內突然需要很大量的 memory, 並且 ruby 不會在用完之後立刻把這些要來的 memory 還給 OS, 所以造成了 memory 不夠的狀況。

#結論
Memory 的問題遇到了就是有點麻煩阿，基本上沒有什麼快速又好的解法，就是要慢慢搞。但如果可以把問題 reproduce 出來應該就解了一半。

Papertrail 的 log 嚴格來說並不是很精準，原因是在 heroku 的 addon 上面的話，只有 method, url 而沒有更進一步的 request 資訊。這個部份我有試著做了一個 [rack middleware 叫 log_spy](https://github.com/mz026/log_spy)，把每一個 request 的詳細資訊丟上 amazon SQS，讓後續的 service 可以接著把這些 queue 上面的東西存到各種 database 等等。

再者是找到問題的點之後要調整 memory 用量的話，調整的方法就看問題的點是什麼了。但原則上我認為**在問題發生之前，不要太早做各種 optimization 而犧牲了 code 的架構和可讀性**。因為通常在問題發生之前做的 optimization 也都不會是真正的問題點，反而把 code 寫整齊，要找問題或是要修正都會比較容易得多。

> Premature optimization is the root of all evil.


註1：我覺得這個方法真的蠻聰明的，但可惜不是我發明的QQ。這個方法是從[這篇 blog](http://blog.skylight.io/hunting-for-leaks-in-ruby/) 上面來的。
