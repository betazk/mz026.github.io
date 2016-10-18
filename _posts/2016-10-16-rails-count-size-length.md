---
layout: post
title: ActiveRecord count v.s. size v.s length
comments: true
description: "ActiveRecord 的 association 有三個長得很像的 methods: count, size, length. 使用起來的功能差不多，但在 database 的 query 上則是有明確的不同"
tags:
  - rails
  - activerecord
---

`ActiveRecord` 的 association 有三個長得很像的 methods: [`#count`](http://api.rubyonrails.org/classes/ActiveRecord/Associations/CollectionProxy.html#method-i-count), [`#size`](http://api.rubyonrails.org/classes/ActiveRecord/Associations/CollectionProxy.html#method-i-size), [`#length`](http://api.rubyonrails.org/classes/ActiveRecord/Associations/CollectionProxy.html#method-i-length) 。
使用起來的功能差不多，但在 database 的 query 上則是有明確的不同。
如果一不小心有可能會誤中地雷發出意想不到的 query 阿！

[TL;DR;](#section-1)

## 地雷說明

我們有一個 `User` model, 在一個 `user` model 下面會對應到多個 `UserSupportMessage` model (`has_many`)。
在一個 query 裡面，我們希望同時 query 出一組 `user` (`id` 屬於 `user_id_pool`) 和其對應的 `user_support_messages` 的個數 ([註1](#section-2))

```ruby

users = User.where(id: user_id_pool).includes(:user_support_messages)
data = users.map do |user|
  {
    user: user,
    message_count: user.user_support_messages.count
  }
end

```

如果實際把上面的程式拿去執行的話,
會發現 **即使我們已經 eager load `messages` 了，但 `user.messages.count` 還是會發出 database query**
但如果改成用 `user.messages.size` 或者是 `user.messages.length` 則不會。

## Ruby Array methods

會有這樣的誤會，可能要從 Ruby [`Array#count`](http://ruby-doc.org/core-2.2.0/Array.html#method-i-count), [`Array#size`](http://ruby-doc.org/core-2.2.0/Array.html#method-i-size), `Array#length` 談起。
基本上後兩者是完全一樣的 method (alias), 而 `#count` 則是可以接參數做到類似 `select` 的事情 (但是是回傳符合條件的個數而不是 subset)。


那麼在 ActiveRecord 下面的這三個 methods 呢？
我們可以來做一下下面的實驗

---


1) *`ActiveRecord_Associations_CollectionProxy#count`*

```
pry(main)> user = User.first

pry(main)> user.user_support_messages.count
SELECT COUNT(*) FROM "user_support_messages" WHERE ...
=> 1

pry(main)> user.user_support_messages
SELECT "user_support_messages".* FROM "user_support_messages" WHERE ...

  ...

pry(main)> user.user_support_messages.count
SELECT COUNT(*) FROM "user_support_messages" WHERE ...
=> 1

pry(main)> user.user_support_messages.count
SELECT COUNT(*) FROM "user_support_messages" WHERE ...

```

在這邊我們要注意的是，**重覆呼叫 `count` 的時候，不論 association 被 load 下來了沒，一律都會發出 db query 去問 `count(*)`**


---


2) *`ActiveRecord_Associations_CollectionProxy#size`*

```

pry(main)> user.user_support_messages.size
SELECT COUNT(*) FROM "user_support_messages" WHERE ...
=> 1
pry(main)> user.user_support_messages.size
SELECT COUNT(*) FROM "user_support_messages" WHERE ...
=> 1
pry(main)> user.user_support_messages.size
SELECT COUNT(*) FROM "user_support_messages" WHERE ...
=> 1



pry(main)> user.user_support_messages
SELECT "user_support_messages".* FROM "user_support_messages" WHERE ...

  ...

pry(main)> user.user_support_messages.size
=> 1
pry(main)> user.user_support_messages.size
=> 1
```

在這邊我們會發現，**association 被 load 下來之前，基本上 size 的行為和 count 一樣，就是每呼叫一次就會問 db 一次。但如果 association 已經被 load 下來了，那麼 `size` 則不會再另外發出 db query**


---


3) *`ActiveRecord_Associations_CollectionProxy#length`*

```
pry(main)> user.user_support_messages.length
SELECT "user_support_messages".* FROM "user_support_messages" WHERE ...
=> 1
pry(main)> user.user_support_messages.length
=> 1
pry(main)> user.user_support_messages.length
=> 1
```

至於 `length` 的行為，則是**先把 association 所有的欄位都先 load 下來 (而不是只有 query `count`), 再去算個數。所以後續的呼叫是不會另外發出 db query 的。**


## 實驗結果

統合上面的結果，我們可以整理出以下的表格：

| method                              | query 內容 | 行為                                                |
| ---                                 | ---        | ---                                                 |
| `count`                             | `count(*)` | 每一次呼叫都會 query db                             |
| `size` (without association loaded) | `count(*)` | 每一次呼叫都會 query db                             |
| `size` (with association loaded)    | -          | 不會 query db                                       |
| `length`                            | `select *` | 第一次會把 association load 下來，之後不會 query db |

## 結論

`ActiveRecord` 有很多設計看來是故意希望做成像一般的 array 介面一樣，
但說到底，ORM 還是拿來 query db 的，所以使用上如果真的把它當成 array 在用，可能會發生很多可怕的事情。

從程式設計的角度來說，我其實不確定這樣的設計到底是好還是不好，
理想上應該是要從 API 的介面上避免 developer 犯錯的才對。
但從實務上來說，現在 ActiveRecord 也是很難避免，所以就是自己小心一點ㄎㄎ。


### 註1

以這邊 demo 的 usecase, 其實如果直接下 sql query, 應該比用 `count`, `size`, `length` 都來得更簡單~~粗暴~~。
但這樣的作法就又面臨到抽象層的選擇和 code readability 等等的取捨。
但總之又是另一個問題惹。

