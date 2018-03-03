---
layout: post
title: Hello Redux 2/3 Server Rendering
comments: true
tags:
  - react
  - redux
  - "universal-rendering"
---

這幾個月身邊有好多的 projects 開始使用 React, 這一兩個月開始接觸到 [Redux](https://github.com/rackt/redux/)，覺得簡直驚為天人 XD。仔細讀了 Redux 的 source code 就發現他好強阿！覺得好像有更上一層樓的感覺。

這陣子用 Redux 做的東西有：
- 一個 desktop app (with Electron)
- 一個 universal webapp
- 把一個用 [fluxxor](https://github.com/BinaryMuse/fluxxor) 做的 universal webapp 接上 Redux, 讓之後的功能可以用 Redux 做，但又不用把之前的重寫。

我打算把一些東西整理一下寫幾篇文章, 目前預計把 Redux 相關的東西整理成三篇：
- [Hello Redux](http://mz026.logdown.com/posts/332288-hello-redux-1-1-react-redux-guide): 關於 Redux 的一些基本介紹，包括我認知的 redux 的一些概念，和我覺得他設計得很棒的地方。
- Server Rendering: 介紹如何使用 Redux 和 [react-router](https://github.com/rackt/react-router) 做 server rendering
- [Unit Test](http://mz026.logdown.com/posts/313379-hello-redux-3-3-unit-test): 在測 Redux 的 code 的時候我遇到的問題和解決的方法，還有怎麼在測試的時候和 webpack loaders 和平共處。

## 本文來了
我打算從第二個部份 Server Rendering 開始，原因是我覺得 Redux 的介紹應該很多了。然後 Server rendering 這個部份比較有趣一點。

## Server Rendering (universal/isomorphic)?
在講 Server Rendering 之前，我們應該先了解在這個東西出現之前我們遇到了什麼問題。

### Single Page App
當瀏覽器的效能越來越好之後，漸漸地在做網站的時候，越來越多的邏輯從 server side 移到 client side, 也就是我們常聽到的 webapp 或者是 single-page app，javascript 的明星也由早期的 jQuery 變成了 Backbone.js, AngularJS, EmberJS 到最近的 React。

Single Page App 讓網站的角色變成一個獨立的 app，就像是 iOS 或者是 android app 那樣，用 API 來跟 server 溝通。如此一來有很多好處：從 user 的角度切入，user 不再需要一直 reload 網頁，大大增進的 user experience。從 loading 的角度切入，本來集中在 server 的 rendering work 現在移到了每一個 end user 的瀏覽器裡面。從開發流程的角度切入，server 和 client 從此有了統一的介面(API)，不會再因為改動了 database 裡面的一個欄位就讓前端的 code 大壞。前端的 app 變成一個 self-contained 的東西，在維護上也大大地減低了困難。

但是 Single Page App 並不是完全沒有缺點。最主要的問題有二：
- SEO 的困難。因為 single page app 的 html element 都是由前端的 javascript render 出來的，搜尋引擎的 crawer 無法看到這些，所以對於 crawer 來說，看到的總是一個空空如也的 html body。([但最近這個問題好像有了轉機](http://googlewebmastercentral.blogspot.tw/2015/10/deprecating-our-ajax-crawling-scheme.html))。

- initial loading 很慢。Single Page App 要等 javascript 先下來之後，才由 javascript 再把 html render 出來。所以 user 一開始要等比較久才可以看到網頁的內容。

### 所以我們才要 Server Rendering 阿！
在這邊我們把整個 architecture 分三塊：一塊是 API server 負責提供 data, 再者是 web server 負責和 client share code 和 render html(就是我們下面指的 server), 最後是 client，也就是在瀏覽器裡面執行的 code。

architecture 可以參考 [airbnb 的 blog](http://nerds.airbnb.com/isomorphic-javascript-future-web-apps/)
在這邊先用一張圖來呈現：
<img style="max-width: 100%" class="center" src="http://nerds.airbnb.com/wp-content/uploads/2013/11/Screen-Shot-2013-11-07-at-10.29.32-AM.png" alt="圖片來源：Airbnb (http://nerds.airbnb.com/isomorphic-javascript-future-web-apps/)">
(圖片來源：[airbnb](http://nerds.airbnb.com/isomorphic-javascript-future-web-apps/))

Server Rendering 的意思是說，讓一部份的 code 先在 server 執行，然後當 server 把畫出一開始的畫面所需要的 data 拿到之後，把 initial page 的 html 先 render 好，並且把這包 html 和剛才拿到的 data 打包起來一起送給 client。
當 client 拿到了 initial page html (所以同時解決了 SEO 和一開始畫面要等很久的問題)和這包 data 之後，再接續著剛才 server 完成的進度繼續讓 app 運作下去。這個概念其實相當直覺，但 React 出現之後才重新被認真討論的原因是，因為 React 本身的設計可以讓這件事情可以更優雅地被完成。

簡而言之呢，要做到 Server Rendering 可以分成三個部份：
- 讓 server 拿到 render initial page 所需要的 data
- 利用這些 data 把 html render 出來
- 把這些 data 本身打包送到 client 去

## 終於要進入 Redux 了的主題惹
在下面我不會帶到全部的 code, 但下面的 code 都放在[這邊](https://github.com/mz026/universal-redux-template)

### 情境設定
假設我們現在有一個動態的頁面叫作 `Question`，我們要 call 一個 API 把目前的 `questions` 拿下來然後把他 render 在畫面上。
我們有：
- 一個 `questions` reducer
- 一個 `loadQuestions` action，會在 load 成功之後把 data 放進 `questions` reducer

Redux app 的進入點大概長這樣：

```javascript
ReactDOM.render((
  <Provider store={store}>
    <Question />
  </Provider>
), document.getElementById('root'));
```
`containers/Question.js` 大概長得像這樣：

```javascript
import React, { Component, PropTypes } from 'react';
import { connect } from 'react-redux';
import { loadQuestions } from 'actions/questions';
import _ from 'lodash';

class Question extends Component {
  componentDidMount() {
    this.props.loadQuestions();
  }
  render() {
    return (
      <div>
        <h2>Question</h2>
        {
          _.map(this.props.questions, (q)=> {
            return (
              <p key={q.id}> { q.content }</p>
            );
          })
        }

      </div>
    );
  }
}

function mapStateToProps (state) {
  return { questions: state.questions };
}

export { Question };
export default connect(mapStateToProps, { loadQuestions })(Question);
```

`actions/questions.js` 大概長這樣：

```javascript
export const LOADED_QUESTIONS = 'LOADED_QUESTIONS';
export function loadQuestions() {
  return function(getState, dispatch) {
    request.get('http://localhost:3000/questions')
      .end(function(err, res) {
        if (!err) {
          dispatch({ type: LOADED_QUESTIONS, response: res.body });
        }
      })
  }
}
```
於是當 `Question` mount 之後，就會送一個 API 給 server 把 data 拉下來, 進而 update view 了。

### On Server Side
但是當我們要在 server 上 render 的時候，事情就變得比較複雜了。
我們預期最後做好的樣子是，**server 的架構完成之後，是不需要隨著前端的邏輯改動而改變的。也就是說，這邊的 server 是不知道 component 的 business logic 的**。
回顧一下我們要在 server 上做的三件事：

- 讓 server 拿到 render initial page 所需要的 data
- 利用這些 data 把 html render 出來
- 把這些 data 本身打包送到 client 去

首先要解決的問題是，當一個 request 進來的時候，我們怎麼知道要 call 哪一個 api 或者是要怎麼準備 state 呢？
第二個問題是，我們在 call 了 async 的 API 之後，要怎麼知道什麼時候 data 才準備好了呢？

第一個問題其實是和 routing 有關。以上面的 `Question` component 為例，真正拿 data 的點是在它的 `componentDidMount()` 裡面。在大部份的情況，我們會需要一個 router 來控制 url 和對應 component 的關係。所以我的做法是，**在每個 routing 的 leaf node (如果有巢狀 routing 的話，就是最裡面的那個) 放一個 static method `fetchData()`**, 這樣的話，在 server 我們就可以用 react-router 去 match 最後被 route 到的 component, 進而得到 data 的進入點。

第二個問題我的解法是用 promise, 意思是說，**讓上述的 `fetchData()` return 一個 promise, 當這個 promise resolve 的時候就代表所有 async 的動作都完成了，也就是 data 都 ready 了的意思**。

這個部份在 server 的 code 大概長這樣：
```javascript
import { RoutingContext, match } from 'react-router'
import createMemoryHistory from 'history/lib/createMemoryHistory';
import Promise from 'bluebird';
import Express from 'express';

let server = new Express();

server.get('*', (req, res)=> {
  let history = createMemoryHistory();
  let store = configureStore();

  let routes = crateRoutes(history);

  let location = createLocation(req.url)

  match({ routes, location }, (error, redirectLocation, renderProps) => {
    if (redirectLocation) {
      res.redirect(301, redirectLocation.pathname + redirectLocation.search)
    } else if (error) {
      res.send(500, error.message)
    } else if (renderProps == null) {
      res.send(404, 'Not found')
    } else {
      let [ getCurrentUrl, unsubscribe ] = subscribeUrl();
      let reqUrl = location.pathname + location.search;

      getReduxPromise().then(()=> {
        let reduxState = escape(JSON.stringify(store.getState()));
        let html = ReactDOMServer.renderToString(
          <Provider store={store}>
            { <RoutingContext {...renderProps}/> }
          </Provider>
        );
        res.render('index', { html, reduxState });
      });
      function getReduxPromise () {
        let { query, params } = renderProps;
        let comp = renderProps.components[renderProps.components.length - 1].WrappedComponent;
        let promise = comp.fetchData ?
          comp.fetchData({ query, params, store, history }) :
          Promise.resolve();

        return promise;
      }
    }
  });

});
```
server view template `index.ejs`

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Redux real-world example</title>
  </head>
  <body>
    <div id="root"><%- html %></div>
    <script type="text/javascript" charset="utf-8">
      window.__REDUX_STATE__ = '<%= reduxState %>';
    </script>
    <script src="http://localhost:3001/static/bundle.js"></script>
  </body>
</html>
```

然後在 `containers/Question.js` 裡的加一個 `fetchData()` static method:

```javascript
class Question extends Component {
  static fetchData({ store }) {
    // return a promise here
  }
  // ...
}
```

然後到這裡，我們就遇到了第三個問題…
就是**在 `fetchData()`裡面，我們要怎麼 reuse 本來的 action (`loadQuestions()`) 同時又 return 一個 promise 呢？**

這個問題我的解法是**做一個 middleware, 在這個 middleware 裡面去 call api，然後從這個 middleware return 一個 promise 給 `fetchData()`。**

這個 middleware 大概長這樣：
`middleware/api.js`

```javascript
import { camelizeKeys } from 'humps';
import superAgent from 'superagent';
import Promise from 'bluebird';
import _ from 'lodash';

export const CALL_API = Symbol('CALL_API');

export default store => next => action => {
  if ( ! action[CALL_API] ) {
    return next(action);
  }
  let request = action[CALL_API];
  let { getState } = store;
  let deferred = Promise.defer();
  // handle 401 and auth here
  let { method, url, successType } = request;
  superAgent[method](url)
    .end((err, res)=> {
      if ( !err ) {
        next({
          type: successType,
          response: res.body
        });

        if (_.isFunction(request.afterSuccess)) {
          request.afterSuccess({ getState });
        }

      }
      deferred.resolve();
    });

  return deferred.promise;
};
```

如此一來，本來的 action 就可以改成：
`actions/questions.js`

```javascript
export const LOADED_QUESTIONS = 'LOADED_QUESTIONS';
export function loadQuestions() {
  return {
    [CALL_API]: {
      method: 'get',
      url: 'http://localhost:3000/questions',
      successType: LOADED_QUESTIONS
    }
  };
}
```

然後 `fetchData()` 就變成這樣：
`containers/Question.js`

```javascript
import React, { Component, PropTypes } from 'react';
import { connect } from 'react-redux';
import { loadQuestions } from 'actions/questions';
import _ from 'lodash';

class Question extends Component {
  static fetchData({ store }) {
    return store.dispatch(loadQuestions());
  }
  //...
}
```

然後我們就搞定了，YA!

## 總結
這陣子做 universal rendering 的感想是，universal rendering 的效果確實是蠻不錯的。就是在一開始 load 網頁的時候真的會有 "哇好快！"的感覺。但是另一方面，在開發的時候確實也因為這樣，會要在很多地方多花一些工夫設計和繞開一些東西 (像是 webpack 的各種帥氣 loader 等等)。
我個人的感覺是，綜合各種 trade-off 下來之後，做 universal rendering 還算是一個划算的選擇。

註：上面的 code 可以到[這邊](https://github.com/mz026/universal-redux-template)去看，跑起來之後請找 `http://localhost://3000/q/1/hello`。耶！
