---
layout: post
title: 親愛的，我把 nodejs 測試變快惹 300 倍
comments: true
description: ""
tags:
  - nodejs
  - testing
  - mocha
  - tdd
---

在 Codementor 開發的過程中，"development experience" 是我們一直相當重視的一環。翻成白話文，就是開發起來要爽就對了。
我們前端的 webapp 主要是用 React 來做的。其中 unit-test 是不可或缺的一塊。
隨著時間和功能的演進，整個 project 的程式碼、伴隨著測試的 test case 數量和檔案的量也越來越多。
過程中，我們漸漸發現本來很順暢的開發流程開始變得不順了，有一塊就是 unit-test 的執行速度。

在這篇文章裡，我想把一些我們定義、發現、解決問題的過程記錄下來和大家分享。

## 情境介紹

Codementor 前端主要使用的 framework/library 是 React 佐 webpack 襯 babel, express universal rendering。
其中測試的工具是 mocha。團隊的成員有些習慣 TDD，有些不是。所以跑測試的 pattern 有些是會在短時間內狂跑特定一兩個 test case ([red-green-refactor](http://blog.cleancoder.com/uncle-bob/2014/12/17/TheCyclesOfTDD.html)), 有些則是會重覆跑某個檔案的 test case。

在一開始，由於必需要讓 mocha 執行的程式可以看懂 ES6 和 JSX，所以我們用了[babel 官網](https://babeljs.io/docs/setup) 上面的作法：

```json
// package.json
{
  "scripts": {
    "test": "mocha --compilers js:babel-register"
  }
}
```

再加上一些 [`--watch` 的 option](https://mochajs.org/#usage) 就可以跑一個 mocha 的 proecess 起來，然後開始 watch 測試。

但是慢慢地我們的 test case 和檔案越來越多(目前大約有四千多個 test case)，這個 command 變得越來越慢，到後來甚至要兩分鐘才可以把 process 跑起來。

這樣的執行速度終於讓我們受不了惹，於是我們開始想要怎麼辦。


## 我們到底要的是什麼呢？

當發現 "測試跑起來太慢" 之後，我們開始一層一層去分析這個問題，包括討論大家的工作流程和習慣等等。
目的是既然要處理這個問題，就希望一次把它調整到最好。

- 首先是對測試的期望：

> 我們希望測試**可以在開發的時候快速的給我們 feedback**。

也就是說，在邏輯驗証的部份，理想上是完全不需要 debugger 就可以完成的。
而從這個需求出發，我們就不難發現，其實我們並不是希望整個測試跑起來變得超快，因為在開發的當下，我們通常是只會專注於整個 code base 的一小部份，而不是全部。也就是說，*即使整體的 test case 跑起來並不是那麼快，但當下正在開發的一兩個檔案必須要越快越好。*

- 再來是跑測試的 pattern：

目前團隊裡開發的習慣，有一部份的人是慣用TDD，~~另一部份的人則不是 廢言 XD~~
編輯器方面，有一部份人是使用地表最強編輯器 vim，另一部份的人則是用 sublime。

TDD 和編輯器之所以會影響到跑測試的方法，主要是因為如果是 TDD 的話，那 red-green-refactor 的循環必須是要非常快速的。
換句話說，如果改一行 code 要執行測試的時候要等個五秒，那在等待這五秒的時間當中，思考就被中斷了，根本無法 TDD 阿！
反之，如果不是用 TDD 的話，那對於測試執行的時間可能可以有稍微高一點的容忍度。
因為測試可能是在完成一個相對較大的功能之後才會被執行，這時候測試執行的時間比較不會中斷整個思考的流程。

而編輯器方面，~~則是單純的因為想要宣揚國威XD~~ 則是因為不同的編輯器可能會有不同的 plugin 可以讓我們在編輯器裡面直接執行測試。
像是 vim 就有 [vim-test](https://github.com/janko-m/vim-test)。這些技巧如果熟悉的話，對於開發的速度是會有顯著的差異的。


## 到底本來的作法是慢在哪呢？

其實這樣的執行速度是有點超出我的預期的。因為即使 mocha 不是以執行速度為號召的 test framework，但應該也還不至於慢成這樣。
另一個可能就是要 compile ES6 和 JSX 要用的 babel。
小小實驗一下果然沒錯，如果讓 mocha 使用 babel 的 compiler 的話，光是要執行一個無腦的測試就要 8 秒阿！

```javascript
import { expect } from 'chai'

describe('No brainer test with babel', ()=> {
  it('should pass', ()=> {
    expect(true).to.eq(true)
  })
})
```

```
$ time mocha --compilers js:babel-register src/test/m1-test2.js


  No brainer test with babel
    ✓ should pass


  1 passing (2s)


mocha --compilers js:babel-register   5.50s user 2.51s system 100% cpu 7.998 total
```

下面則是對照組，如果沒有過 babel 的話就只要 0.24 秒：

```javascript
var expect = require('chai').expect

describe('No brainer test without babel', function() {
  it('should pass', function() {
    expect(true).to.eq(true)
  })
})
```

```
$ time mocha src/test/m1-test3.js


  No brainer test without babel
    ✓ should pass


  1 passing (9ms)

mocha src/test/m1-test3.js  0.22s user 0.02s system 100% cpu 0.240 total
```

上面的結果並不是要說 babel 有多慢之類的，而是要釐清問題的核心。
如果說我們的目的是讓開發的流程更順暢，那在這個情境下 babel 的 compile 時間應該是最主要的問題，
而不是一些不是瓶頸卻可能很費工的東西，例如說把測試裡面不小心在可以用 shallow rendering 卻用了 deep rendering 的地方全部改掉。

## 解決流程

在發現問題之後，接下來就是要想辦法讓 babel register 變快一點。
直覺想到的就是先找看看有沒有 cache 的機制，但是找來找去好像找不到阿阿阿！
(有看到 [build 的 cache](https://babeljs.io/docs/usage/babel-register/#environment-variables-babel-cache-path), 但就是找不到 register 用的 QQ)
(如果有誰找到請跟我說！)


在 babel register 無法加速的前題下，另一個方向則是：

**在不用 compile ES6/JSX 的狀態下，mocha 的速度其實是很可以接受的。**
那我們如果先把 ES6/JSX 都先 build 成 ES5，然後就讓 mocha 去跑這些預先 build 好的檔案，
這樣的速度是可以接受的嗎？

於是我們做了一個小實驗，基本上就是手動先把 application code/test code 全都 build 好，
然後在沒有透過 babel register 的狀態下跑一次，結果發現速度快了很多。
尤其是只跑一兩個檔案甚至一兩個 test case 的情境下，更是完全沒有問題。

而這樣的話，那問題就變成了：每次都要 build 全部的檔案很慢。

但這個問題就很容易解決了，
我們土砲了一個簡單的 cache 機制，
其實實際上在這個情境裡面，我們需要的並不是多高深或精準的東西，只是要讓

1. build 過的東西不要重覆再跑
2. 新的檔案或者是有改動的檔案要重 build：這部份 babel-cli 本來就有提供了


在經過一番實做之後，最後的流程如下：

- 先把所有的檔案 build 過一次，放在另一個暫存資料夾裡面(好比說 `.babel-test-build`)。並且同時把 application code 的檔案路徑和 md5 的值對應記錄下來變成一個 cache record
- 然後再用 `--watch` 的 option 執行 babel, 並且 `--skip-initial-build`

這時候我們會有一個 watch process 在執行，每當 application code 有改動的時候，`.babel-test-build` 裡面對應的 ES5 就會立刻被更新。
因為這時候改動的檔案數很少(因為我們理論上同一個瞬間只會編輯一個檔案)，所以 `.babel-test-build` 裡面的更新是可以很快的。
而這時候，我們就可以執行這個檔案的測試，不管是從編輯器裡跑或是從另一個 terminal 跑，都可以享有"不透過 babel register" 的執行速度。

當然，這中間有蠻多細節在裡面的，像是[pre build 的過程](https://github.com/mz026/universal-redux-template/blob/master/bin/test-prebuild)、[路徑的對應](https://github.com/mz026/universal-redux-template/blob/master/bin/test)、[編輯器的設定等等](https://github.com/mz026/universal-redux-template/blob/master/editor_configs/vimrc)，但這些直接看 code 應該比較快，在這邊就不再多說。



## 結論

在這次的過程中發現的東西是，**去優化開發的流程和體驗其實是 CP 值很高的事情**。
除了讓每個開發者少等一些時間之外，

> 更重要的是減去這些等待的時間，可以讓我們的思考流程不會被中斷。

當思考的流程是連成一氣的時候，很多邏輯上的錯誤都能因此避免，這個才是更珍貴的地方。

另外，去分析團隊裡成員的工作方式也是很值得做的一件事情。以這次的情境來說，
它讓我們避免了用錯誤的方向去解決問題。某種程度有點像是 "developer 的 user study"，
在精準的分析之後，才可以解決到真正要被解決的問題。
