---
layout: post
title: 你今天 code review 了嗎？
comments: true
description: "在 Codementor，我們大約在一年多前認真把 code review 正式加入開發流程的一環。但過程當然不是一帆風順的。
在這篇文章裡面，想要和大家分享我們對 code review 的認識、我們的作法和解決問題的過程。"
tags:
  - "code-review"
  - "pull-request"
  - programming
  - codementor
---

Code review 和 unit test，早睡早起一樣，都是屬於"大家都覺得要做，但是常常做不到" 的事情之一。
在 Codementor，我們大約在一年多前認真把 code review 正式加入開發流程的一環。但過程當然不是一帆風順的。
在這篇文章裡面，想要和大家分享我們對 code review 的認識、我們的作法和解決問題的過程。


## 什麼是 code review, 為什麼我們要 code review？

在大部份的軟體團隊裡面，code review 是相當重要的一環。根據 [wiki](https://en.wikipedia.org/wiki/Code_review) 上面的定義：

> Code review is systematic examination (sometimes referred to as peer review) of computer source code. It is intended to find mistakes overlooked in the initial development phase, improving the overall quality of software.

它的目的是某種程度上預防可能的錯誤，和增進程式的品質。
而除此之外，透過適當的 code review 更可以讓團隊裡面的每一位成員了解各個部份的程式碼。
如此一來，在維護同一份程式的時候，團隊的成員就更能對程式的架構有一定的共識。可以避免不小心重覆做了其實已經有的功能，或者是做了一個功能結果發現和本來設計的理念大相逕庭的狀況發生。

當團隊的成員們對於不管是自己或別人寫的 code 都有一定程度的了解之後，互相維護、修改彼此的程式碼就變得比較容易，團隊的人力分配彈性就變大了。並且也可以在心態上，把"人" 和"code" 拆開，避免 "這邊是我寫的，除了我之外，誰也別想碰！" 的狀況。

另外，當 code review 變成一個正式的流程之後，我們在寫任何一行code的時候，就會清楚地知道 "這每一行等下都會有人來看"。這個時候羞恥心就立刻提高。於是乎，寫程式的時候就會更加嚴謹、comment 掉的 code 就會刪掉、本來想要複製貼上的時候就會認真 refactor。於是乎，程式的品質就提升了。

再者，code review 是一個讓大家可以討論程式架構或細節的情境。有了這樣的流程之後，可以讓團隊的成員們很習慣討論並且藉此修改程式。我覺得這是一件很珍貴的事情，因為**好的程式永遠不會是被寫出來的，而是被改出來的。**


## 用講的比較容易阿！

當然，code review 有一百種好處。但如果實際上做起來那麼容易的話，那 [legacy code](/20161210/working-happily-with-legacy-code) 也不會出現了。

{: .center}
![did-you-code-review-today--cant](/images/did-you-code-review-today/cant.jpg)

接下來會以我們本身團隊的經驗，和大家分享怎麼開始、維持 code review 💪


## 文化

> 文化永遠凌駕於任何技巧和流程之上的東西。

**當我們在進行 code review 的時候，其實說白了，就是要開始挑戰別人和被別人挑戰。**
因為難以避免地，在 code review 的時候我們會提出 "我認為這邊的作法還可以更好，可能要改成如何如何"，而被 review 的人可能也會提出 "我覺得本來的作法比較適合，因為如何如何" 等等。基本上是一個不斷挑戰對方和辯論的過程。而程式的架構，其實也會在這樣不斷辯論的激烈過程中被精煉成最完美的狀態。

但這時候，溝通的態度就非常關鍵了。有可能同樣一個問題，會因為討論的態度甚至是語氣的不同，而從客觀健康的討論變成意識型態的無謂賭氣。
而溝能的能力是一個需要練習的技能，在每個人都變成溝通之王之前，好的文化可以確保整個過程會是在一個愉快的狀態下進行。

這樣的文化可能可以由很多種方式被建立。

> 對我們來說，這樣的文化是基於"信任"。

我們信任團隊的彼此都是認真希望程式變好，所以不會因為懶惰、不在乎而寫出爛 code。所以如果看到程式有可以更好的地方，那必定是有什麼地方不小心疏忽了或是沒有想到有更好的作法。

我們信任團隊的彼此都是客觀理性的，所以如果有人對我的程式提出什麼建議，那並不是針對我個人，而是他有考慮到什麼我沒有考慮到的地方。

在這樣的情境下，大部份的討論都可以理性而舒服的進行。
當然，要維持這樣的文化的具體作法則是會深入到每一個日常生活的細節。包括討論的時候的語氣，留 review note 的時候的用字遣詞等等。

並且，**文化是會互相感染的。**在一個很信任且互相尊重的環境下，一個本來可能很刻薄的人也會收斂起來。反之，在一個互相猜忌、責怪對方的團隊下，可能很少人能做到就事論事。

(關於文化，我覺得 CircleCI 的[這篇文章](https://circleci.com/blog/kindness-is-underrated/)寫得很棒)


## 萬事起頭難

上面的情境都是當 code review 這樣的流程和文化都被建立好了的時候。

*那如果我們的團隊就是沒有 code review 這樣的流程該怎麼開始呢？*

我覺得這時候可以分成兩種狀況：

- 團隊的成員或老闆覺得不需要這樣的東西
- 大家覺得需要這樣的東西，但不知道怎麼做


如果是第一種狀況的話，那這時候我們要做的事情就是想辦法讓他們知道 code review 的好處。
大部份的時候，覺得不需要的原因都是來自於進行 code review 有一定的時間成本，但 code quality、人力分配的彈性、文化什麼的卻都是相對隱性且長期的好處，很難在短期立刻被量化來衡量。

因為我們已經知道，長久下來，有(被有效率執行的) code review 之後，整體的工程效率是會比沒有 code review 還要大很多的。
所以在策略上，可以試著**先想辦法讓 code review 某種程度被有效執行，然後隨之而來的產出提升就可以讓大家沒有它也不行惹。**

在執行上，可以試看看先從一兩個工程師開始。或者是從一部份的比較複雜的功能開始 code review。簡單的說，就是先用比較低成本的方法試試水溫，而不是一次整個撩落去。這樣的話，可以透過這些嘗試讓團隊感覺一下 "有 code review" 的感覺。另一方面，也可以針對流程上彼此磿合出最適合團隊的作法。

而當大家慢慢的了解 "阿！原來有 code review 之後世界這麼美好阿！" 之後，就可以把它正式的加入開發流程了。


而如果是第二種狀況的話，其實沒有標準的答案。簡單來說就是要試各種辦法，這點在下面會跟大家分享我們自己的經驗。

## 具體作法們

關於要如何進行 code review，可能有一百種作法。結論是每一種作法都有其好壞，而且每個團隊，甚至同一個團隊在不同階段適合的方法也都不同。
**所以不論採用哪樣的作法，團隊的成員定期檢討目前的流程進而作出調整是必須的。**

以 Codementor 來說，我們一開始的作法是搭配 git 和 Github(或 Bitbucket) 來做：

- 每一個改動要 merge 進主要的 branch 之前，都要透過 pull request
- 上面的 pull request 要包括 unit test
- 然後任意請一到兩個同事進行 code review。當然，如果改動的部份比較大，那可能會特別請當初設計那部份架構的同事一起 review


這個流程跑了一下下之後，我們就發現一個主要的挑戰：

就是常常 pull request 一發出去結果超大。這樣子 reviewer 必須得特別空出一大段時間來看這個大的 pull request。
而我們在工作的時候，很難隨時空出一大段時間。於是 review 的時間點通常會很慢。
再者，如果一個 pull request 太大，那如果經過 review 之後發現有重大的架構要修改（雖然如果有發生這件事，那通常代表 review 是有用的，因為不然這個可能歪掉的架構就直接被 merge 了而且沒有人知道 XD）那本來做的人就要改動很多東西，這樣也造成了時程很難控制的問題。

### 把大的 feature 切小

但是一個 pull request 如果要切小的話，到底要多小才夠小呢？
我們的作法是**儘量控制一個 pull request 可以讓 reviewer 在 15 分鐘之內看完。** 這樣一來，reviewer 就可以在平常寫程式到一個段落的時候，把 review pull request 當作一個休息和緩衝。並且如果在 review 過後有什麼需要大改的話，因為每個 pull request 都相對小，所以改動起來也不會多花太多時間。

**要把一個完整的 feature 切成很多個 15 分鐘內可以看完的 pull request 其實是需要練習甚至討論的。**
但在這樣的過程裡面，其實剛好也迫使我們在實作大型的、複雜的東西之前，先定義一個架構出來。這樣不管是對於程式的設計上或時程預估都會有很正向的幫助。並且以我們的習慣，過了 review 的 code 就會隨時被推到 production。所以這樣把 feature 切小的流程，也可以大幅減少了真正 integrate 到 production 時候的風險。

舉例來說，如果我們要做[這個 profile 的頁面](https://www.codementor.io/mz026)，以它功能的量和複雜度而言，是不可能在 15 分鐘之內 review 完的。
這時候我們會試圖找到它最 top level 的架構，然後從 top level 做到細節。以這個例子來說，第一個 pull request 可能是只各個 tab 和它們對應的 route，而每個 tab 點進去都只是一個空白的頁面。在 review 完這樣的架構之後，我們會用類似 feature flag 的方式把這樣的東西直接推到 production，順便確定它在 top level 的架構沒有問題。接下來我們可能會再根據 priority，把每個 tab 下面的各式細節分別實作上去。

大致上要把大的 feature 切小的撇步有：

- 從 top level 開始，再進入細節。
- feature flag 或類似讓我們可以把半成品 release 到 production 的東西是我們的好朋友。

當然，要把 feature 切小塊有時候並不是件簡單的事，但花點工夫去做這件事會是很划算的。

### 用各式小工具來自動化

另外，在這樣的流程下，我們之前常常碰到的問題是：

- 有人發 pull request 給我了，但我不知道
- 有人在我的 pull request 上面留了 review note, 但我不知道

所以時常發生了問來問去或等來等去的狀況。針對這個，我們接了一些 github 的 webhook 到 slack bot 上面，並且做了一些小工具可以讓這整個流程更順暢。詳情可以[看這邊](https://www.codementor.io/mz026/dont-miss-another-pull-request-6wsl2dzbp) XD

### 小結

- 進行 code review 的作法有 n 種，要找到最適合自己的團隊的那種。
- 作法合不合適，是會隨著時間而改變的。團隊的成員們同時也要留意是不是有什麼地方可以改進。
- 把 feature 切小塊，對所有的事情都有幫助。
- 有各種工具（或自己做XD）可以讓這一切變得更順暢。

## code review 的時候該看什麼呢？

當我們定義好了流程之後，下一步就是該認真 review code 了。
但是看到 github 的介面紅紅綠綠的 diff，要從哪邊看、看什麼東西才能更的效率呢？

簡單的說，就是 **首先我們要知道這個 pull request 是做什麼的。然後看該看的，不要看不該看的**


### 知道這個 pull request 是做什麼的

在 review 一個 pull request 的時候，**reviewer 有一定的情境認識是很重要的。**
簡單的說就是他要知道這個 pull request 到底要幹麻。通常一些基本的說明，或是 issue/trello card/wireframe 就可以很有幫助。

### 看該看的

什麼是該看的呢？問這個問題之前，我們應該先問我們自己，code review 的目的是什麼呢？
**針對程式本身來說，code review 的主要目的是確認 merge 進去主要 branch 的 code 都有一定的品質。**
從這個角度出發，我們在 review 的時候，應該最先注意那些 "和程式品質有直接相關的東西"。


#### 1. 從較高的抽象層看到較低的

以 backend 來說，我們目前用 Rails 再搭配 [service object](https://hackernoon.com/service-objects-in-ruby-on-rails-and-you-79ca8a1c946e) 來建立我們的 feature。這時候首先要看的東西就是這些 service object。它們被放在哪一個 namespace、怎麼命名。這點意味著我們怎麼定義這個 service 和其他 service 的關係。在現有的程式下，這樣的定義有符合整體設計的概念嗎？

再確定了 service 的定義沒有問題之後，接下來才進到每個 service 裡面去看它的邏輯是不是正確，或者 db query 有沒有什麼誤會等等。

而以 front end 來說，目前我們使用的是 React/Redux。在這樣的環境下，redux container 的 data 交換和 store 裡面定義的 data structure 就是最核心的部份，接下來才是 component 的配製和實作的細節。

#### 2. 留意閱讀的體驗

**在從高的抽象層看到低的抽象層的過程中，你可以很清楚的知道實作的脈絡嗎？**

如果不行的話，那就要想一下是不是有什麼地方可以調整得更好。大部份的時候，精準的命名可以解決很多誤會。但是說真的，[命名絕對不是一件簡單的事](https://martinfowler.com/bliki/TwoHardThings.html)。因為精準的命名背後代表的事對整個 domain model 有很深刻的了解。如果找不到一個好的命名方式，或者說不清楚這個東西為什麼要這樣命名，可能要再認真想一下，這塊程式究竟打算做什麼事情。

但不管怎樣，如果從頭到尾看下來，你可以很清楚地了解這個改動的作法，那基本上應該沒什麼大問題了才對。

### 不要看不該看的

什麼是不該看的呢？應該說是"人類不該看的" XD

其實就是那些可以被自動化的東西。像是各式 [linter](http://eslint.org/) 和 CI 可以幫我們省去很多機械化的東西。
簡單的說就是，我們的時間很寶貴，儘可能的自動化一些東西吧！


## 結論

Code review 如果可以有效執行的話，會是在很多地方都很有幫助的一件事情。
但 "有效執行" 這件事並不是隨隨便便就可以做到。它需要很多嘗試並且要隨時調整。
而在流程之上，文化更是扮演了核心的角色。


## 工商服務：全球最大的工程師教學平台上線囉！

如果你看了這篇文章，覺得心有戚戚焉，很喜歡 [Codementor](https://www.codementor.io) 的文化和作法，想要打造一個世界一流的工程師平台，讓世界看到台灣，歡迎加入我們！
如果你會寫 Ruby on Rails，或者會寫 React/Redux，我們需要你阿阿阿阿！歡迎來[跟我們聊聊](mailto:hello+codereviewblog@codementor.io)！
