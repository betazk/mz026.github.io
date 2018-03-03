---
layout: post
title: Redux middleware Functional Programming
comments: true
description: "最近做了一些跟 React Redux 有關的 projects。Redux 的設計有加入了很多 functional programming 的概念在裡面，其中我覺得最優雅的是 middleware 的設計和作法。在讀了它的 source code 之後，決定寫一篇關於它的文章。"
tags:
  - react
  - redux
  - middleware
  - "functional-programming"
---

最近做了一些跟 React Redux 有關的 projects。Redux 的設計有加入了很多 functional programming 的概念在裡面，其中我覺得最優雅的是 middleware 的設計和作法。在讀了它的 source code 之後，決定寫一篇關於它的文章。

這篇文章會假設大家都已經對於 React, Redux, Redux middleware 的用法和想要解決的問題都有一定程度的了解。如果還有不了解的地方，建議可以先看一下[之前的這篇介紹 Redux 的文章](http://mz026.logdown.com/posts/332288-hello-redux-1-1-react-redux-guide)喔 :D

在這篇文章裡面，我想要帶大家走過 Redux middleware 的 source code，並且從中討論關於 functional programming 的部份。

## Functional Programming...?

關於 javascript 究竟算不算是一個 functional programming 的語言，可能會有很多不同的看法。我認為，從[定義上的 functional programming](https://en.wikipedia.org/wiki/Functional_programming) 來說，它可能算不上是一個 funcational language。畢竟它沒有天生的 immutable data，也沒有處處充滿 functional language 常見的遞迴。**但是以 "function 是 first class citizen" 和 [closure](https://en.wikipedia.org/wiki/Closure_(computer_programming)) 來看，它倒是很 functional 沒錯。**

## Redux Middleware

什麼是 redux middleware 呢？
**Redux 的 middleware 是介於 `store.dispatch` 和 `reducers` 之間的中間人。更精準地說，我們 call 的 `store.dispatch()` 其實是由一層一層的 middleware 所組成，到最後一層才會進到 `reducer`**。可以用下面這張圖來表示：

<img class="center" src="http://user-image.logdown.io/user/8925/blog/8821/post/332288/NqajldzQTpWbXR3YjEyN_Redux-middleware.jpg">

其中每一個 middleware 的工作都是**根據某些條件，動態地修改/取消 `action`**，也就是說，當一個 `action` 進到一個 middleware 的時候，

- 它有可能依照現有的 state(`store.getState()`) 被修改
- 有可能被 pass 到下一個 middleware，或者不被 pass
- 當所有 middleware 都走過之後，最後會進到 reducer

## 一個 middleware 長成怎樣呢？
在定義一個 middleware 的時候，它的 function signature 長成這樣：

```javascript
({dispatch, getState}) => next => action => {
  //middleware content here
}
```

這個很多箭頭的東西是 ES6 的語法，如果用 ES5 寫的話是這樣：

```javascript
function(store) {
  return function(next) {
    return function(action) {
      var dispatch = store.dispatch;
      var getState = store.getState;
      //middleware content here
    }
  }
}
```

**用這樣的作法，我們可以利用 javascript 的 [closure](https://en.wikipedia.org/wiki/Closure_(computer_programming))，把 `{getState, dispatch}`, `next` pass 進去，進而讓 middleware 可以拿到這些東西。**

用這樣一層一層的作法有什麼用意呢？答案是

1. **我們可以在 application 的一開始(設定階段) 就先把 `store`(更準確的說是`getState` 和 `dispatch`) 丟進去，而不用在每次執行 `dispatch` 的時候把 `store` 傳來傳去。**
2. **把 `next` 用上面的結構傳進去，可以讓 middleware 被以 `compose` 的方式串起來** (下面會說明)


## 實際上用起來是長怎樣呢？
我們從 middleware 的使用端這邊切入會比較好了解整體的架構。
在 Redux 裡面，我們把 middleware 掛上去 store 的作法是這樣：

```javascript
// configureStore.js
import { applyMiddleware, createStore } from 'redux'

return createStore(
  rootReducer,
  initialState,
  applyMiddleware(middleware1, middleware2)
)
```

首先遇到的是 [`createStore`](http://redux.js.org/docs/api/createStore.html):

## `createStore(reducer, [initialState], [enhancer])`:
`createStore` 這個 function 接的參數分別是

- reducer: reducer 是控制 store 如何改變的入口，詳細說明可以看[這邊](http://mz026.logdown.com/posts/332288-hello-redux-1-1-react-redux-guide)
- initialState: 在整個 redux app 要被叫起來的時候，我們在某些情境下會需要這個 app 從某一個特定的 state 開始運作，像是[universal(isomorphic) rendering](http://mz026.logdown.com/posts/308147-hello-redux-2-3-server-rendering)的時候
- enhancer: 是和 middleware 相關的部份。**enhancer 的長相(signature)就是 "接一個 `createStore` 的 function, 然後 return 一個和 `createStore` 一樣介面的 function"**。**事實上我們透過 `applyMiddleware` return 出來的結果就是一個 enhancer。**利用不同的 `enhancer`，我們可以改變 store 的行為。事實上 **`applyMiddleware` 所產生出來的 `enhancer` 就是改變了 `store.dispatch` 的行為。**

我們如果仔細看 `createStore` 的程式碼的話，會發現這段：

```javascript
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }

    return enhancer(createStore)(reducer, initialState)
  }
```

也就是說如果我們有把 enhancer 傳進 createStore 裡面的話，那基本上得到的 store 就會是透過 enhancer 傳出來的結果。

## `applyMiddleware`

很明顯的，把 middleware 連起來的工作是由 `applyMiddleware` 這個 function 完成的。`applyMiddleware` 所回傳出來的結果就是上述的 `enhancer`。source code 如下：

```javascript
// src/appliyMiddleware.js
export default function applyMiddleware(...middlewares) {
  return (createStore) => (reducer, initialState, enhancer) => {
    var store = createStore(reducer, initialState, enhancer)
    var dispatch = store.dispatch
    var chain = []

    var middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```

在上面的 `configureStore` 裡面，我們把 `middleware1`, `middleware2` 傳進 `applyMiddleware`。

```javascript
applyMiddleware(middleware1, middleware2)
```

而接下來我們就可以好好看看在這個 enhancer 裡面，我們是怎樣利用 `middleware1`, `middleware2` 去改變 `dispatch` 的行為

首先，我們先用"最原始的 `createStore` 建出一個 store"，這個 store 是還沒被 enhancer 改變過的，我們在下面稱它作"原始的 store":

```javascript
var store = createStore(reducer, initialState, enhancer)
```

接著，我們宣告一些 local variables, 其中的 `middlewareAPI` 有兩個 key:

- `getState`: 這是 "原始的store" 的 `getState`, 事實上在後面我們會發現，在被 enhancer 改變的過程中，store.getState 其實是不受影響的。
- `dispatch`: 這是我們企圖利用 `applyMiddlware` 來改變的部份。注意他是一個 function 包著外面的 local variable `dispatch`, 但這個 local variable 會隨著程式的進行而改變。就是下面的這行：

```javascript
 dispatch = compose(...chain)(store.dispatch)
```

接著，我們用上面的 `middlwareAPI` map 出 `chain` 這個 local variable，根據上面 middleware 的定義，這個 `chain` 其實長得大概像這樣：

```javascript
chain = [
  function middlewareCreator1(next) {
    // with getState, dispath
  },
  function middlewareCreator2(next) {
    // with getState, dispath
  },
  ...
]
```
**在 chain 的每一個 element 裡面，都可以 access 到 `middlewareAPI` 的 `getState` 和 `dispatch`**

在 map 出 `chain` 之後，我們先把些"接 `next`" 的 function 們 `compose` 起來, 再把本來最原始的 `store.dispath`(還沒被 enhancer 改變過的版本) 丟進去，並且 assign 給 `dispatch` 這個 local variable 讓 `middlewareAPI.dispatch` 可以拿到這個新版的 dispatch：

```javascript
  dispatch = compose(...chain)(store.dispatch)
```

這一行其實做了蠻多事情的，我們可以把它拆開來看：

- `var comopsed = comopse(...chain)`
- `dispatch = composed(store.dispatch)`

## `compose(...chain)`

`compose` 做的事情是把傳進來的多個 function 串起來, 簡而言之大樣是這樣：

```javascript
compose(f, g, h)
```

在結果上相當於

```javasript
function(...args) {
  f(g(h(...args)))
}
```

如下圖所示：
<img class="center" src="http://user-image.logdown.io/user/8925/blog/8821/post/702274/lrhK8dLyTvKFKmjSx1mg_compose%20(1).jpg" alt="compose (1).jpg">

我們可以把每一個 function 想像成一個盒子，把一個球(input)丟進去這個盒子之後，會吐出另一個球(output)。由於我們要把多個盒子(function)串起來，也就是**說從第一個盒子吐出來的球，會被丟進第二個盒子作為input**，所以每個盒子的input 和 output 必需要有同樣的格式。而 `compose` 做的事情，就是把多個盒子拼裝起來變成一個大的盒子，所以這個大的盒子從 input/output 的格式來看，是會和每一個小盒子是一樣的。也就是我們可以把一個球丟進這個大盒子，這個大盒子會吐出另一個球。

有了上面 `compose` 的概念之後，我們可以回頭來看我們的 middleware chain。在上面的 `chain` 裡面，**每一個盒子(middlewareCreator)的 input 都是一個叫 `next` 的 function，每一個盒子的 output 也會是一個 function, 用來丟進下一個 middleware 作為 `next`。這正是 redux middleware 運作的方式：在每一個 middleware 裡面可以選擇性地呼叫下一個 middleware(`next`)。**

以上面的盒子、球的觀念來比喻的話，每一個middleware creator(`function(next){...}`)就是我們說的"盒子"，而實際上對每一個 action 進行操作的 function (`function(action){...}`) 就是我們所說的"球"。在下面的程式裡

```javascript
var comopsed = comopse(...chain)
dispatch = composed(store.dispatch)
```

我們把第一顆球 `store.dispatch` (它會把 `action` 丟給 `reducer`) 丟進 `compose` 過後的大盒子裡面，得到一個被層層 middleware 包裝過的大球。然後把這個大球作為新版的 `dispatch` 放回 store 裡面：

```javascript
return {
      ...store,
      dispatch
    }
```

這樣一來，我們在 application 裡面所呼叫的 `store.dispatch` 就會變成這個被包裝過的大球。這也就是為什麼我們在 application 裡面 `dispatch` 一個 `action`，它會先經過層層 `middleware` 才到達 `reducer` 的原因。

耶！講完惹！

## 結語

對我自己來說，在接觸到 Redux 之後，覺得對於 Javascript 這個語言有了更進一步的了解和認識，其中很大的一部份就是來自於它和 functional programming 的關係。在 application 架構的設計上，運用 closure 可以把一些 dependencies 藏起來，讓程式的其他部份變得更單純。

這種方式的缺點可能就是程式讀起來要花比較多一點時間去習慣和了解，但以一個 library/framework 的角度出發，如果這樣可以換來更單純的 application code，我想是非常值得的。

而且這樣寫起來漂釀又有趣阿！
