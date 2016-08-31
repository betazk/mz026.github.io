---
layout: post
title: Rails Race Condition Test With RSpec
comments: true
description: "這陣子在寫 Rails 的時候，陸續遇到了 race condition 的狀況。在解決它的過程中，也同時想要用 unit test 把它保護起來。
於是有一些心得記下來和大家分享"
tags:
  - rails
  - activerecord
  - rspec
  - testing
  - "race-condition"
---



這陣子在寫 Rails 的時候，陸續遇到了 race condition 的狀況。在解決它的過程中，也同時想要用 unit test 把它保護起來。
於是有一些心得記下來和大家分享。


## 哪一種 Race Condition?

根據 [wiki 上的定義](https://en.wikipedia.org/wiki/Race_condition)，race condition 是指 **某一種行為的結果會根據執行的時間順序或其他不可控制的因素而影響結果。**

> A race condition or race hazard is the behavior of an electronic, software or other system where the output is dependent on the sequence or timing of other uncontrollable events. It becomes a bug when events do not happen in the order the programmer intended. The term originates with the idea of two signals racing each other to influence the output first.

在這篇文章要討論的是 server application 最常見的 model validation 的 race condition。



## 前情提要

在 Rails 的 application 裡面，當我們要寫入(新增/修改)一個 model 的時候，通常都會在 application layer 用 [validator](http://guides.rubyonrails.org/active_record_validations.html) 來確保寫入的值是合法的。好比說 `User#name` 必需要存在，`User#email` 必需要是 case insensitive unique 等等。而同時在 persistent layer(database)，如果可以的話我們也會加上對應的限制來確保我們寫入 database 裡面的東西不會髒掉。以述的例子的話，如果是 relation database 我們可能會指定 `User#name`　這個欄位是 `NOT NULL`，然後在 email 的欄位加上 unique constraint。

在這樣的設定之下，如果說我們企圖寫入不合法的 data 的時候，在企圖寫入 database 之前，就會被 application 的 validator 擋下來。這時候 application 就可以做相對應的處置。好比說如果我們有一個 user email 已經被用走了，而如果我們又嘗試要用同一個 email 建立一筆新的 user record 的時候, application 就會噴出 exception：

```ruby
existing_user = User.create!(email: 'the-email@test.com', name: 'jack')

duplicated_one = User.create!(email: 'the-email@test.com', name: 'mary') # => ActiveRecord::RecordInvalid
```



## 情境設定

在我們的 server application 裡面，我們利用 [JWT](https://jwt.io/) 來做 [Single Sign On(SSO)](https://en.wikipedia.org/wiki/Single_sign-on)，於是在 server 上我們會在每個需要 authenticate 的 request 做下面的事情：

- 檢查 user 有沒有合法的 JWT?
- 沒有
  - 回 `401`
- 有
  - database 裡有沒有這個 user?
    - 有: 用這個 user 來執行後續的 API
    - 沒有: 立刻用 JWT 裡面的資訊在 database 建一個 user record, 然後用這個 user 來執行後續的 API


看起來很正常對吧？

在 "找 user 或建一個新的 user" 這件事情上面，我們把它抽象成一個 [service](https://blog.engineyard.com/2014/keeping-your-rails-controllers-dry-with-services): `FindOrCreateUser`

```ruby
class AuthService::FindOrCreateUser < Service
  attr_reader :user_attr

  def initialize user_attr:
    @user_attr = user_attr.with_indifferent_access
  end

  def perform
    if existing
      existing.update(attr)
      existing
    else
      User.create!(attr)
    end
  end

  def existing
    # find user with @user_attr
  end
  private :existing
end
```

上面是簡化版本的 `FindOrCreateUser` service。理論上 JWT 裡面帶的資訊是從我們自己的 user service 帶出來的 (而不是 user input)，所以我們預其它都會是合法的格式。於是在這個 service 裡面我們並不 handle `User::create!` 有可能會丟出來的 `ActiveRecord::RecordInvalid` 這個 exception。
或者說，我們希望如果我們自己帶的 data 格式有什麼誤會的話，這個 exception 直接噴到 error console 上面。

把這個 service 做出來之後，實際上使用也都相安無事 ~~呃可以轉台了~~ 。但推上 production 之後卻開始在[監控 error 的服務](https://raygun.com/) 上面發現隨機的 500。訊息的內容則是 `ActiveRecord::RecordNotUnique`， 寫著：


```
PG::UniqueViolation: ERROR: duplicate key value violates unique constraint...DETAIL: Key (username)=(xxxx) already exists
```

(我們的 user table 上面有一個 unique field 叫 `username`)。

這時候我們眉頭一皺，發現案情並不單純。
因為理論上如果有存在的 `username` 的話，那在 `existing` 這個 method 裡面應該就要被找到了才對，
並且我們實際上用一個已經存在的 `username` 去找的話，也可以有正確的行為。

這是什麼巫術阿~~~

## Why? (Debug)

在一番掙扎之後，發現因為**前端的 web app 有可能在很短的時間之內，連續發出 request 到 server**。這時候可以看成有一個以上的 request 幾乎是同時被處理的。
假設我們有兩個 request 同時到達 server，這時候對於兩個 request 來說，`existing` 的檢查：

```ruby
if existing
  ...
else
  # both requests would enter this line!
end
```

都會是 `nil`，因為當這兩個 request 同時被執行的時候，我們確實還沒有寫入任何東進 database。
於是這兩個 request 都認為自己可以寫一個新的 user record 進 database，所以後面寫入的那個 request 就會噴出上面的 excpetion 惹。


## How To Test it?

在發現 bug 的原因之後，下一步就是先把相對應的 test case 補起來。
因為本來的 test case 並沒有 handle 到這樣的 race condition，
**所以首先我們必須要先補一個 test case, 並且在這樣的 test case 裡面 reproduce 這個情形才可以。**

在下面的 test case 裡面，我們是用 [RSpec](http://rspec.info/) 來實作，但理論上用其他的 testing framework 應該也是差不多。
要 reproduce 實際上發生的狀況，有一個關鍵是 "兩個 request 同時進來" 這件事情。
於是在 test case 裡面，我們開兩個 thread 去模擬這個狀況。

```ruby
# find_or_create_user_spec.rb

it 'handles race condition' do
  threads = 2.times.map do
    Thread.new do
      service.perform
    end
  end

  threads.each(&:join)
end

```

一跑下去，結果喔喔喔，果然重現惹：

```
1) AuthService::FindOrCreateUser#perform message handles race condition
     Failure/Error: user = User.create!(attr)

     ActiveRecord::RecordInvalid:
       Validation failed: Username has already been taken
     # ./app/services/auth_service/find_or_create_user.rb:20:in `perform'
     # ./spec/services/auth_service/find_or_create_user_spec.rb:32:in `block (6 levels) in <top (required)>'
```

在 test case 可以重現 error 之後，剩下的東西就相對單純了。
這時候我們在 service 裡面安排一個 `retry`，基本上就是說，如果撞到了就先等一下下再做。
因為我們預期等一下下之後，再重做一次的時候，另一個 request 已經把 record 寫進去了，所以剛才的 `existing` 的條件就可以被看到了。

```ruby
  def perform
    if existing
      existing.update(attr)
      existing
    else
      user = User.create!(attr)
      user
    end
  rescue ActiveRecord::RecordInvalid, ActiveRecord::RecordNotUnique => e
    if @retried
      raise ArgumentError, e.message
    else
      @retried = true
      sleep 0.3
      retry
    end
  end
```



## RSpec, Database cleaner setup

在加入了這個 test case 並且改了 service 之後，這個 test case 也順利過關了。
但這時候卻發現了另一個問題，
就是其他本來沒問題的好多 test case 卻出現了重覆 record 的錯誤。
並且這些出錯的 test case 有各式各樣，彼此之間並沒有什麼邏輯的關聯。
所以這時候合理懷疑是剛才加入的 test case 搞的鬼。


在又經過一番~~罵髒話~~苦戰之後，發現原來跟 [database_cleaner](https://github.com/DatabaseCleaner/database_cleaner) 的設定有關。

**在測試的時候，我們希望每一個 test case 跑的時候，database 裡面都是沒有任何 record 的。**

為了要做到這件事情，我們利用 [database_cleaner](https://github.com/DatabaseCleaner/database_cleaner) 幫我們：

1. 在全部的 test case 之前，先把所有 table truncate 掉。
2. 在之後的每一個 test case 之前，都先進一個 transaction，然後在每一個 test case 之後，再 rollback 這個 transaction

藉由這樣，我們就可以確保每一個 test case 在跑的時候，database 裡面都是乾乾淨淨的。
這樣也可以同時確保我們每一個 test case 之間沒有相依性，
不會有這個 test case 一定要在另一個 test case 之前跑才會過這種事情發生。

但當我們用兩個 thread 來測剛才的 race condition 的時候，看來是另一個 thread 所建立的 connection 無法正確的被 rollback 回來。
所以導致當這個 test case 跑完之後，database 裡面殘留了一個 user record, 於是其他的 test case 就因此跑出了奇怪的錯誤。

要解決這件事情，我們的作法企圖讓某一些特定的 test case 用 truncate 的方法來清掉 database(而不是 trasaction)
至於為什麼不全部的 test case 都用 truncate 就好了呢？答案是因為 transaction 比較快。

設定的方式大致上是利用 [rspec 的 user-defined meta data](https://www.relishapp.com/rspec/rspec-core/docs/metadata/user-defined-metadata):

```ruby
# spec/rails_helper.rb

config.around(:each) do |example|
  if example.metadata[:db_cleaner_strategy] == :truncation
    DatabaseCleaner.strategy = :truncation
  else
    DatabaseCleaner.strategy = :transaction
  end

  DatabaseCleaner.cleaning do
    example.run
  end
end
```

然後再把相對應的 test case 加上 meta data:

```ruby
context "message", db_cleaner_strategy: :truncation do
  it 'handles race condition' do
    require 'user'
    threads = 2.times.map do
      Thread.new do
        service.perform
      end
    end

    threads.each(&:join)
  end
end
```

就大功告成啦！爽喔！

## Conclusion

Race condition 相關的錯誤基本上就是屬於"有時候有有時候沒有"的一種 bug，簡單的說就是比較麻煩處理。
但處理的過程中，其實會對於整份 code 和相對應的工具(像是 RSpec, database_cleaner)的行為更加了解。
基本上是一件"你不會希望天天遇到，但遇到了也是還有點有趣"的一件事情。

Happy Hacking!!




