---
layout: post
title: Rails with_connection, reap 與 connection leak
comments: true
description: "在這篇文章裡面，我想分享的是 ActiveRecord connection pool 的 leak 和 reap 的故事"
tags:
  - rails
  - thread
  - with_connection
  - "connection-leak"
  - database
  - ActiveRecord
---

在之前遇到[疑似 rails connection leak](/20160917/rails-connection-leak)之後，就一直大概知道 [ActiveRecord 的 `with_connection`](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/ConnectionPool.html#method-i-with_connection) 可以避免 connection leak。前陣子在設定 RabbitMQ 的時候，出現了 connection pool timeout 的 error，在 debug 的時候才發現原來我好像一直沒有親眼看過 Rails leak connection 的時候會長得怎樣 XD

所以我試圖用一些方式讓 Rails leak database connection，發現好像沒有很單純阿。
在這篇文章想要把一些嘗試的過程記錄下來和大家分享。

關於 connection leak 和 connection pool 的介紹可以看[這邊](/20160917/rails-connection-leak#hello-connection-leak)。


## 重現問題

在另外開的 thread 裡面，如果有使用到 connection 而忘記 checkin 回去的話，理論上這個 connection 就會 leak 掉。

*但是要怎麼重現這樣的狀況呢？*

下面是概略的環境設定：

- Rails 4.2.8
- 首先我試著把測試的環境建立成一個 rake job
- 並且把 connection pool 的 size 調成 3


```ruby
task :test_conn => :environment do
  pool = ActiveRecord::Base.connection_pool

  threads = 3.times.map do
    Thread.new do
      pool.checkout
    end
  end

  threads.each(&:join)

  pool.checkout
end
```

在這邊，我開了 3 個 threads，並且在每一個 thread 裡面 checkout 一個 connection 出來。
在確定 connection 都被 checkout 出來了 (`threads.each(&:join)`) 之後，
理論上因為所有pool裡的 connection (3個) 都 leak 掉了，所以當我們要再 checkout 一個 connection 出來的時候，應該要出現 timeout 的 error。

但這個 rake job 跑下去的時候，*完全沒有任何的 error 出現 QQ*

後來發現，connection 有一個 method 叫 `in_use?` 可以來幫我們確認 connection 有沒有正在被使用。
當 connection 沒有在被使用的時候，他會回傳 `nil`，而當正在被使用的時候，他會回傳正在使用的那個 thread。

於是把剛才的 rake job 加上一些 debug 的資訊變成這樣：

```ruby
def log_connections conns
  conns.each do |idx, conn|
    puts "#{idx}: #{conn.object_id}; #{conn.in_use?}"
  end
end

task :test_conn_w_log => :environment do
  pool = ActiveRecord::Base.connection_pool
  conns = {}

  threads = 3.times.map do |i|
    Thread.new(i) do |index|
      conns[index] = pool.checkout
    end
  end

  threads.each(&:join)
  puts "after joining"
  log_connections(conns)

  pool.checkout
  puts "after re-checkout"
  log_connections(conns)
end
```

得到的 output 是這樣：

```
after joining
0: 47195627544380; #<Thread:0x0055d92d3bd1a0>
1: 47195627073520; #<Thread:0x0055d92d3bc408>
2: 47195626859620; #<Thread:0x0055d92d367d90>

after re-checkout
0: 47195627544380; #<Thread:0x0055d92952a258>
1: 47195627073520;
2: 47195626859620;
```

我們發現在 join 之後，也就是在 thread 已經結束了之後，connection 都還是在 "使用中" 的狀態。
然後在這樣的狀況下 checkout，不僅沒有問題，反而其他的 connection 還變成 "非使用中" 的狀態了。

在實際看了 [ActiveRecord connection pool 的 source code](https://github.com/rails/rails/blob/v4.2.8/activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb) 之後, 發現在[每次企圖 checkout](https://github.com/rails/rails/blob/v4.2.8/activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb#L418-L427) 的時候，如果 pool 裡面沒有可用的 connection 了，ActiveRecord 會試圖做一個叫作 [reap](https://github.com/rails/rails/blob/v4.2.8/activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb#L390-L407) 的行為，這個動作會把 "現在在 '使用中' 但 owner 已經死掉" 的那些 connection 回收起來。
這也就是為什麼在重新 checkout 之後，另外兩個 connection 又變成 "非使用中" 的狀態了。

但是如果在執行 reap 的時候，所有正在使用的 connection 的 owner 都還沒死，那這時候 reap 就無法回收任何 connection。
即使在 timeout 的限制（5秒）之內，有其他的 thread 已經用完 connection 了，這些用完的 connection 還是無法被使用。

```ruby
task :test_conn_cannot_reap => :environment do
  pool = ActiveRecord::Base.connection_pool
  conns = {}

  threads = 4.times.map do |i|
    Thread.new(i) do |index|
      conns[index] = pool.checkout
      puts "got connection in thread #{index}"
      sleep 1
    end
  end

  threads.each(&:join)
  puts "after joining"
  log_connections(conns)

  pool.checkout
  puts "after re-checkout"
  log_connections(conns)
end
```

這樣得到的 output 會是：

```
got connection in thread 0
got connection in thread 1
got connection in thread 2
rake aborted!
ActiveRecord::ConnectionTimeoutError: could not obtain a database connection within 5.000 seconds (waited 5.000 seconds)
```


但是這個時候, 如果配合上上面的 `with_connection` 就沒問題了：

```ruby
task :test_conn_w_with_connection => :environment do
  pool = ActiveRecord::Base.connection_pool
  conns = {}

  threads = 4.times.map do |i|
    Thread.new(i) do |index|
      pool.with_connection do |conn|
        conns[index] = conn
        puts "got connection in thread #{index}"
        sleep 1
      end
    end
  end

  threads.each(&:join)
  puts "after joining"
  log_connections(conns)

  pool.checkout
  puts "after re-checkout"
  log_connections(conns)
end
```

output:

```
got connection in thread 0
got connection in thread 2
got connection in thread 1
got connection in thread 3
after joining
0: 47069386019060;
2: 47069385769900;
1: 47069385570280;
3: 47069386019060;
after re-checkout
0: 47069386019060;
2: 47069385769900; #<Thread:0x00559e5ee72258>
1: 47069385570280;
3: 47069386019060;
```


## 結論

Connection count 的限制是在各種時候都必須要小心的事情。再加上 thread 這類有"時間"性質的變因加入之後，會讓整個東西變得更複雜。
基本上我個人是比較偏好在寫 Rails 的時候，讓 library/framework 這層去控制 thread 比較令人安心一點。
也就是說，**如果可以讓 thread 的複雜度不要隨著 application feature 的增加而增加會是比較好的狀況**。
