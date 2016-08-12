---
layout: post
title: ActiveRecord::Relation#find v.s Array#find
comments: true
description: "ActiveRecord::Relation#find v.s Array#find"
tags:
  - rails
  - activerecord
  - backend
---

ActiveRecord query 出來的結果，是一個 `ActiveRecord::Relation` 的 instance。基本上 `ActiveRecord::Relation` 有很多和 `Array` 長得很像的 methods。
其實我不確定這樣的設計到底是好還是不好；

在使用上，某些情境下是可以把 query 出來的結果直接拿來像 array 一樣操作，好比說：

```ruby
name_of_users = User.where('age > ?', 30).map do |user|
  user.name
end
```

但是這樣的介面設計有時候會讓人忽略了，其實它的背後是有可能再送 database query 出去的。

今天遇到了一個狀況是關於 `#find` 這個 method。

# ActiveRecord::Relation#find v.s Array#find

`ActiveRecord::Relation` 和 `Array` 這兩個 class 都有 `find` 這個 method [(註1)](#section)。直覺上兩者的差異就是前面的應該是會送 database query 出去而後者不會，而實際上也是這樣 ~(可以轉台惹)~

但比較討厭的地方就是有些時候你以為 ActiveRecord 不會送了，但他還會!

好比說我們有一個 user pool:

```
pry(main)> user_pool = User.where(id: [1,2,3,4,5])
DEBUG -- :   User Load (3.4ms)  SELECT "users".* FROM "users" WHERE "users"."id" IN (1, 2, 3, 4, 5)

pry(main)> user_pool.class
=> User::ActiveRecord_Relation
```

到目前為止都還很OK，我們送出了 database query, 但接下來：

```
pry(main)> user_pool.find(1)
DEBUG -- :   User Load (1.0ms)  SELECT  "users".* FROM "users" WHERE "users"."id" IN (1, 2, 3, 4, 5) AND "users"."id" = $1 LIMIT 1  [["id", 1]]

pry(main)> user_pool.find(2)
DEBUG -- :   User Load (1.0ms)  SELECT  "users".* FROM "users" WHERE "users"."id" IN (1, 2, 3, 4, 5) AND "users"."id" = $1 LIMIT 1  [["id", 2]]
```

嗯…怎麼有點怪怪 der，明明剛才在拿 `user_pool` 的時候不就已經送出 database query 了嗎？為什麼 `find` 還會再送一次!!

那如果我們在拿 `user_pool` 的時候，先把它用 `all` 都拿下來呢？

```
pry(main)> user_pool_all = User.where(id: [1,2,3,4,5]).all
DEBUG -- :   User Load (1.3ms)  SELECT "users".* FROM "users" WHERE "users"."id" IN (1, 2, 3, 4, 5)

pry(main)> user_pool_all.class
=> User::ActiveRecord_Relation

pry(main)> user_pool_all.find(1)
DEBUG -- :   User Load (1.1ms)  SELECT  "users".* FROM "users" WHERE "users"."id" IN (1, 2, 3, 4, 5) AND "users"."id" = $1 LIMIT 1  [["id", 1]]
```

結果還是一樣阿靠!! 是說這步的結果是很可以預期的，因為 `user_pool_all` 的 class 也是 `ActiveRecord::Relation` [註2](#section-1)。

那如果把 query 出來的東西轉成 `Array` 呢？

```
pry(main)> user_pool_array = User.where(id: [1,2,3,4,5]).to_a
DEBUG -- :   User Load (1.3ms)  SELECT "users".* FROM "users" WHERE "users"."id" IN (1, 2, 3, 4, 5)

pry(main)> user_pool_array.class
=> Array

pry(main)> user_pool_array.find(1)
=> #<Enumerator: ...>
```

呃…怎麼好像有點怪怪 der
看了 [doc](http://ruby-doc.org/core-2.3.1/Enumerable.html#method-i-find) 之後才發現，原來是 `Enumerable#find` 的介面不一樣阿!

```
pry(main)> user_pool_array.find {|u| u.id == 1}
=> #<User:0x007f33ee27f438>
```

喔喔喔! 用 `Array` 去找就不會再發 database query 出去了。但這好像也是廢話，都轉成 Array 了再發 qeury 出去也太過份了吧！

那如果我們用 `Enumerable#find` 的介面去 call `ActiveRecord::Relation` 呢？

```
pry(main)> user_pool.find {|u| u.id == 1}
=> #<User:0x007f33efae0388>
```

什麼!!這是什麼巫術!!
實際上殺進 [source code](https://github.com/rails/rails/blob/v4.2.7.1/activerecord/lib/active_record/relation/finder_methods.rb#L67-L73) 一看，才發現原來如果給了 block 的話，就會先轉成 array。原乃奴此! [(註3)](#section-2)

```ruby
def find(*args)
  if block_given?
    to_a.find(*args) { |*block_args| yield(*block_args) }
  else
    find_with_ids(*args)
  end
end
```


### 註1

`find` 這個 method, 其實是分被定義在 [`ActiveRecord::FinderMethods`](http://api.rubyonrails.org/classes/ActiveRecord/FinderMethods.html#method-i-find) 和 [`Enumerable`](http://ruby-doc.org/core-2.3.1/Enumerable.html#method-i-find) 上面


### 註2

其實嚴格來說，`user_pool` 和 `user_pool_all` 的 class 並不是 `ActiveRecord::Relation`，而是它的 sub-class `User::ActiveRecord_Relation`。但它是繼承了 `ActiveRecord::Relation` 沒錯：

```
pry(main)> user_pool.class.ancestors
=> [User::ActiveRecord_Relation,
 ActiveRecord::Delegation::ClassSpecificRelation,
 ActiveRecord::Relation,
 ...
]
```

### 註3
這是 Rails 4.2.x 的行為，我不確定到 [Rails 5](https://github.com/rails/rails/blob/v5.0.0.1/activerecord/lib/active_record/relation/finder_methods.rb#L64-L67) 會不會有所改變。

