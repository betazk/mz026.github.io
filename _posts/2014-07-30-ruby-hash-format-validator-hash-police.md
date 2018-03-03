---
layout: post
title: 檢查 hash format 小幫手之 Hash Police
comments: true
description: "Hash Police 是 Ruby 上檢查 Hash 格式的小幫手"
tags:
  - hash_police
  - hash
  - validator
  - ruby
---

在寫 Rails test 的時候，時不時就會要檢查 hash 的格式對不對，
好比說有一個 API 是會要 return

```json
{
  "id": 123,
  "nickname": "Michael Jackson",
  "age": 50,
  "alive": false,
  "genres": [ "pop", "funk" ]
}
```

這個格式如果不小心被改壞了，那接這個 API 的 client 就有可能會壞掉。
於是在 controller test 裡面就會出現： (using [RSpec](https://github.com/rspec/rspec-core))

```ruby
response_body_hash = JSON.parse(response.body)
response_body_hash.should have_key('id')
response_body_hash['id'].should be_kind_of(Fixnum)
response_body_hash.should have_key('nickname')
response_body_hash['nickname'].should be_kind_of(String)
# balah balah balah...
```

如果我們的 API 有好多個 keys, 那簡直是檢查不完，而且 code 的可讀性也大大降低。
時間久了就會產生一種想吐的感覺。

## `Hash Police` is at your service

於是我寫了一個叫作 [hash_police](https://github.com/mz026/hash_police) 的 ruby gem,
功能就是你給他一個符合格式的 hash,
他就幫你 recursively 檢查格式有沒有錯。

所以上面的 code 就會變成：

```ruby
response_body_hash = JSON.parse(response.body)
rule = {
  :name => '',
  :age => 1,
  :alive => true,
  :genres => ['']
}
police = HashPolice::Police.new(rule)
result = police.check(response_body_hash)

puts result.error_messages unless result.passed?
result.passed?.should be_true
```

## 還是太醜

沒錯，上面的作法雖然可以不用一個一個 key 去檢查，
但是如果錯了的話，沒辦法好好的用 `RSpec` 的格式 log 出來，
所以搭配[RSpec custom matcher](https://gist.github.com/mz026/9087682)服用的話：
```ruby
RSpec::Matchers.define :have_the_same_format_as do |expected|
  result = nil

  match do |actual|
    police = HashPolice::Police.new(expected)
    result = police.check(actual)
    result.passed?
  end

  failure_message_for_should do |actual|
    result.error_messages
  end
end


actual.should have_the_same_format_as(expected)
```

本來的 test case 就可以寫成：

```ruby
response_body_hash = JSON.parse(response.body)
rule = {
  :name => '',
  :age => 1,
  :alive => true,
  :genres => ['']
}
respose_body_hash.should have_the_same_format_as(rule)
```
這樣就爽快多惹。

### Notes:
- 如果有什麼 bug 或者任何問題，請不吝指教 :D

