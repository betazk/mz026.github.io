---
layout: post
title: Hello Redux 3/3 Unit Test
comments: true
tags:
  - react
  - redux
  - "unit-test"
---

最近陸續用 Redux 做了[一些](https://www.codementor.io/hire) [projects](https://www.refactor.io)，想要把一些心得整理出來。

目前預計整理三篇：
- [Hello Redux](http://mz026.logdown.com/posts/332288-hello-redux-1-1-react-redux-guide): 關於 Redux 的一些基本介紹，包括我認知的 redux 的一些概念，和我覺得他設計得很棒的地方。
- [Server Rendering](http://mz026.logdown.com/posts/308147-hello-redux-2-3-server-rendering): 介紹如何使用 Redux 和 [react-router](https://github.com/rackt/react-router) 做 server rendering (ya! 寫好惹！)
- Unit Test: 在測 Redux 的 code 的時候我遇到的問題和解決的方法，還有怎麼在測試的時候和 webpack loaders 和平共處。

這篇文章是這個系列的第三部份，Unit Test。

## 關於 Unit Test
Unit Test 整個主題其實涵蓋的主題很廣；從為什麼要做 unit test，到測試的環境設定，到把程式的架構設計成可以被測試的狀態等。在大部份的狀況下，testing code 會要根據 production code 選用的 library 或 framework 找到合適的點來切入。好比說我們的 app 如果是用 React + Redux 寫的，那要測這個 app 的做法和要測一個 AngularJS 的 app 的方法就會有很大的不同。

接下來我會大致上介紹一些測試的基本概念，然後花比較多篇幅在 Redux 和 React 相關的東西上面。

## 什麼是 Unit Test? 和其他種類的測試差在哪？
Unit Test 是自動化測試裡面最貼近程式的一種。意思是說，unit test 的測試對象通常是一個 class, 一個 function 或者一個 React component 等可以用 code 來切開的單位；用另一個角度來說，unit test 是給 developer 看的。
如果用比較高階的 acceptance test 來比較的話，acceptance test 就離程式碼本身比較遠一點，acceptance test 的測試對象可能是一個功能 (例如說 signup 的能功，從按按鈕到建立一個 user 等等)。也可以說，acceptance test 是寫給~~用嘴巴~~不會寫 code 的人看的。(像是 PM XD)

舉例來說，一個 unit test 可能是：
```
確保 questionReducer 在接到 QUESTION_LOADED 事件的時候，可以 return 一個新的 question 作為 state
```

而一個 acceptance test 可能是：
```
確保當 user 按下 question link 的時候，他會被帶到 question 的頁面並且看到 render 好的 question 內容
```

## 為什麼要寫 unit test?

**在 unit test 裡面，我們的目的是要確保我們寫的 code 和我們預期的行為一樣。**

當同一份 code 越長越大，越來越多人加入一起開發之後，我們幾乎不太可能再用純人工去檢查每一個 function, 或每個 class 都如我們預期一般的運作。這時候如果有一個自動的測試幫我們在每次改動的時候，重覆地去檢查並且立刻回報給我們知道，那就可以大大地降低我們 debug 的時間。這代表參與開發的每一個人，都可以很大膽地去改動任何東西，因為只要 test 都跑過了，那幾乎就代表其他的東西沒有被不小心改壞。
這意味著 developer 可以放心地在任何時間去 refactor 程式的架構，久而久之就會形成一個良性的循環，讓這份 code 變得越來越穩定，也因為架構變好了，要加新功能或修改也可以再更短的時間內完成。總之就是好處多多潮爽der。

另外一層用意就是，**測試是不會說謊的 document**。
相信大家都有發生兩個星期之後回來看 code, 結果一邊看一邊罵說這誰寫的鬼，結果 `git blame` 一下發現靠腰是自己寫的。
<img class="center" src="https://scontent-tpe1-1.xx.fbcdn.net/hphotos-xta1/v/t1.0-9/11230107_500377160130121_6886935443576714133_n.jpg?oh=04c0a16eebc94f78f9c9cd8152ad8deb&amp;oe=56B72C5D">

**Unit test 這時候扮演的角色就是，當你忘了這個 function 是幹麻用的，或者忘了要怎麼用它的時候，你可以看一下 testing code 他就會 demo 給你看。**

## 說好的 Redux 呢？

要在一個 redux app 加上測試，大概可以分成幾個部份：
1. 選一個 testing framework 和 assertion, mocking 的 library。 像是 `mocha`, `jasmine` 等等，並關把一些相關的設定弄好。
3. 實際開始寫測試，這個部份又可以切成：
  - Action Test
  - Reducer Test
  - middleware Test
  - Component Test
4. 該如何面對 webpack

在下面的介紹我會略過第一部份的細節，詳細的設定可以在[這邊](https://github.com/mz026/universal-redux-template)找到。並且假設大家對於 mocha 和 chai 的 api 都有基本的認識。

## 選一個好用的 testing framework 和設定它
我選了 mocha + chai 的組合，然後讓測試在 nodejs 的環境下執行。在這之前我有試過用 karma 當作 test runner 讓測試跑在瀏覽器上面，最後把 karma 捨棄的原因是因為 webpack 每次的 build 還是有點慢，讓我要 TDD 的時候變得很痛苦。這個故事就比較長一點，之後有機會再說。

在下面的 code 裡面，我選用的 stack 有：
- [mocha](https://mochajs.org/)
- [chai](http://chaijs.com/) (assertion library)
- [sinon](http://sinonjs.org/) (mocking library)
- [Rewire](https://github.com/speedskater/babel-plugin-rewire)

## 該來寫實際的測試了吧
還沒 XD
在寫測試之前，我們可以把這個過程切成幾個步驟：
- 確定要測試的對象
- 確定要測試的行為
- 在測試的環境裡面(`setup`) 去執行上面的那個行為 (`execute`)
- 驗証結果跟我們預期的一樣 (`verify`)

在下面我不會帶到全部的 code, 但下面的 code 都放在[這邊](https://github.com/mz026/universal-redux-template)

## Action Test
在 Redux 的設計下，action 其實相對單純。從外面看 action 的話，**action 的行為其實只有 return 一個 action object 而已**。

假設我們有一個 `actions/questions.js` 長這樣，然後我們想測試他的 `loadQuestions` 的行為：

```javascript
import { CALL_API } from 'middleware/api';

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
套上上述的步驟：
- 確定要測試的對象: `question action creator`
- 確定要測試的行為:
	- `loadQuestions()` 會回傳某個帶有 `CALL_API` key 的 object，裡面有我們預期的內容

我們的測試會長成這樣：
`spec/actions/questions.test.js`
```javascript
import { CALL_API } from 'middleware/api';

// setup
import * as actionCreator from 'actions/questions';
import * as ActionType from 'actions/questions';

describe('Action::Question', function(){
  describe('#loadQuestions()', function(){
    it('returns action `CALL_API` info', function(){
      // execute
      let action = actionCreator.loadQuestions();

      // verify
      expect(action[CALL_API]).to.deep.equal({
        method: 'get',
        url: 'http://localhost:3000/questions',
        successType: ActionType.LOADED_QUESTIONS
      });
    });
  });
});

```

## Reducer Test
Redux 的 reducer 扮演的角色是一個 function，它接原來的 state 和一個 action，然後 return 新的 state。意思是說， **reducer 是根據現有的 state 和接到的 action，來決定這個 state 要什麼樣的改變。**

假設我們有一個 reducer `reducers/questions.js` 長這樣：
```javascript
import * as ActionType from 'actions/questions';

function questionsReducer (state = [], action) {
  switch(action.type) {
    case ActionType.LOADED_QUESTIONS:
      return action.response;
      break;
    default:
      return state;
  }
}

export default questionsReducer;
```

套上上述的步驟：
- 確定要測試的對象: `question reducer`
- 確定要測試的行為:
	- 在接到 `LOADED_QUESTIONS` 的 action 的時候，會把 `action.response` 當成新的 state
  - 遇到不認識的 action type 的時候，會回傳空的 array

 於是我們得到的 reducer 的 testing `spec/reducers/questions.test.js`:

```javascript

import questionReducer from 'reducers/questions';
import * as ActionType from 'actions/questions';

describe('Reducer::Question', function(){
  it('returns an empty array as default state', function(){
    // setup
    let action = { type: 'unknown' };

    // execute
    let newState = questionReducer(undefined, { type: 'unknown' });

    // verify
    expect(newState).to.deep.equal([]);
  });

  describe('on LOADED_QUESTIONS', function(){
    it('returns the `response` in given action', function(){
      // setup
      let action = {
        type: ActionType.LOADED_QUESTIONS,
        response: { responseKey: 'responseVal' }
      };

      // execute
      let newState = questionReducer(undefined, action);

      // verify
      expect(newState).to.deep.equal(action.response);
    });
  });
});
```

## Middleware test:
Middlware 在 Redux 裡面負責的行為是，在 "action 被 `dispatch` 出去之後，在到達 reducer 之前" 這個過程中去攔截 action，進而改變本來 action 原有的行為。Middleware 本身是一個 function，他的 signature 長這樣：

```javascript
function(store) {
  return function(next) {
    return function(action) {
      // middleware behavior...
    };
  };
}
```

如果用 ES6 的語法來表達，看起來會稍微乾淨一點，但基本上他的本體還一樣複雜：
```javascript
store => next => action => {
  // middleware behavior...
}
```

這個部份我覺得是 Redux 設計得最帥氣的部份之一。之後在 Redux 的介紹文章會詳細說明。現在還是先讓我們先搞定測試再說。
假設我們有一個 api 的 middleware `middleware/api.js` 長這樣：
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
  let { method, url, successType } = request;
  superAgent[method](url)
    .end((err, res)=> {
      if ( !err ) {
        next({
          type: successType,
          response: res.body
        });
      }
      deferred.resolve();
    });

  return deferred.promise;
};
```
這個 middleware 做的事情是：

- 攔截有 `CALL_API` 這個 key 的 action,
- 然後根據這個 `CALL_API` 的 value (假設它叫 `request`) 裡面的 `url`，`method` 去發一個 http api call 給 server。
- 當這個 api call 成功之後，再 `dispatch` 一個 `request.successType` 的 action 出去。
- 這個 middleware 本身會 return 一個 promise, 這個 promise 會在 api call 成功之後被 resolve
(比較完整的版本應該要有相對應的 error handling 才對, 但在這邊為了讓事情單純一點就先省略)

如果套上上面的步驟的話就會變成：
- 確定要測試的對象: `api middleware`
- 確定要測試的行為:
	- middleware 會放過沒有 `CALL_API` 的 action
  - middleware 會根據 `action[CALL_API]` 去送一個 api call 給 server
  - middleware 在 request 成功之後，會`dispatch` 一個 `action[CALL_API].successType` 的 event
  - middleware 在 request 成功之後，會 resolve middleware return 的 promise

所以我們的 test code 就會長這樣 `spec/middleware/api.test.js`:

```javascript
import nock from 'nock';
import apiMiddleware, { CALL_API } from 'middleware/api';

describe('Middleware::Api', function(){
  let store, next;
  let action;
  let successType = 'ON_SUCCESS';
  let url = 'http://the-url/path';

  beforeEach(function(){
    store = {};
    next = sinon.stub();
    action = {
      [CALL_API]: {
        method: 'get',
        url,
        successType
      }
    };
  });

  describe('when action is without CALL_API', function(){
    it('passes the action to next middleware', function(){
      action = { type: 'not-CALL_API' };
      apiMiddleware(store)(next)(action);
      expect(next).to.have.been.calledWith(action);
    });
  });

  describe('when action is with `CALL_API`', function(){
    let nockScope;
    beforeEach(function(){
      nockScope = nock(`http://the-url`)
                    .get('/path');
    });
    afterEach(function(){
      nock.cleanAll();
    });
    it('sends request to `path` with query and body', function(){
      nockScope = nockScope.reply(200, { status: 'ok' });

      apiMiddleware(store)(next)(action);

      nockScope.done();
    });

    it('resolves returned promise when response when success', function(){
      nockScope = nockScope.reply(200, { status: 'ok' });

      let promise = apiMiddleware(store)(next)(action);

      return expect(promise).to.be.fulfilled;
    });
    it('dispatch successType with response when success', function(done){
      nockScope = nockScope.reply(200, { status: 'ok' });
      let promise = apiMiddleware(store)(next)(action);

      promise.then(()=> {
        expect(next).to.have.been.calledWith({
          type: successType,
          response: { status: 'ok' }
        });
        done();
      });
    });
  });
});
```
這邊的測試行為比之前的複雜一些, 其中 `nock` 是一個用來測試 nodejs 上面 http request 行為的一個 library, 這個部份有點超過這篇文章的 scope 所以在這邊就假設大家都很熟悉 XD。除了 `nock` 之外，還有幾個可以詳細說明的點：

首先是 test 的巢狀 `describe` 和 `beforeEach`，這樣的做法可以讓每一個 `describe` 下面的內容有一個 context，再者也可以利用在 `describe` function 下面的 local variable, 讓同一個 `describe` 下面的測試可以 share 同樣的 variable。(像是 `nockScope` 就只有 `when action is with CALL_API` 這個 `describe` block 下面的 code 可以 access 到)

再來是**要怎麼在測試的環境裡執行 middleware 呢？**
因為 middleware 本身是一個 function，只是被包了很多層而已，所以要執行它的方法就是一層一層地去 call 他：
```
apiMiddleware(store)(next)(action);
```

最後是 `dispatch successType with response when success` 這邊有 async(非同步) 的行為。Mocha 最基本的行為是**不會**等 async 的 code 執行完的。也就是說，在 async 的 code 被執行到之前，mocha 的 test case (`it()`)就會先結束掉。這樣我們就會無法在 async 的行為完成之後再驗証一些行為。這個部份 mocha 的做法是在 `it` 後面跟的 function 帶一個參數(`done`)，而 mocha 會等到這個 `done` 在 test case 裡面被執行到之後才結束這個 test case。如此一來，我們就可以在 async 的動作執行完之後再呼叫 `done`，這樣就可以確保 mocha 會在我們預期的時間點才結束 test case 了。

## Component Test
在 Redux 下面，我們把 Ract 的 Component 分成兩種：一種是 smart component, 另一種是 ~~stupid~~dumb component。**Smart component 指的是和 Redux 有連結的那些 component, 而 dumb component 則是完全根據 props 來動作的那些。**
Dumb Component 的測試嚴格來說和 Redux 就比較沒有直接的關係了。因為它們就是一般的 React component。Component 的測試常常會和 mock 有關係，所以就在這邊作一個基本的介紹。

Smart component 在實作上面，其實和 dump component 差不多，只是多了 `connect` 的行為，
1. 把 store 的 state 利用一個 function (`mapStateToProps`) 選擇一部份需要用到的 keys inject 進去變成 component 的 props
2. 把部份的 action 也 inject 進去變成 component 的 props

在測試 smart component 的時候，根據 [Redux document](http://redux.js.org/docs/recipes/WritingTests.html) 上面的建議是，繞過 `connect` 的行為，直接測 component 的部份。
假設我們的 comopnent 長成這樣： `containers/Question.js`

```javascript
import React, { Component, PropTypes } from 'react';
import { connect } from 'react-redux';
import { loadQuestions } from 'actions/questions';
import { Link } from 'react-router';
import _ from 'lodash';

class Question extends Component {
  static fetchData({ store }) {
    return store.dispatch(loadQuestions());
  }

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

        <Link to="/">Back to Home</Link>
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

在這個 component 裡面，我們把 `Question` 特別 `export` 出來，這樣的話我們在測試的時候，就可以繞過 `connect` 完之後的 smart component, 而單純地測試 `Question`。

一樣套上上面的步驟：

- 確定要測試的對象: `Qestion Component`
- 確定要測試的行為:
	- `component` 會根據 `props.questions` render 出對應的 elements
  - `component` 會 render 出一個指向 `/` 的 `Link`

於是我們的測試會長成這樣： `spec/containers/Question.test.js`

```javascript
import Container, { Question } from 'containers/Question';
import React from 'react';
import TestUtils from 'react-addons-test-utils';

describe.only('Container::Question', function(){
  let props;
  let Link;
  beforeEach(function(){
    props = {
      loadQuestions: sinon.stub(),
      questions: [
        { id: 1, content: 'question content 1' },
        { id: 2, content: 'question content 1' }
      ]
    };
    Link = React.createClass({
      render() {
        return (<div>MOCK COMPONENT CLASS</div>)
      }
    });
    Container.__Rewire__('Link', Link);
  });

  it('renders questions according to `props.questions`', function(){
    let doc = TestUtils.renderIntoDocument(<Question {...props} />);
    let questionElements = TestUtils.scryRenderedDOMComponentsWithTag(doc, 'p');

    expect(questionElements.length).to.equal(props.questions.length);
  });
  it('renders a link back to `/`', function(){
    let doc = TestUtils.renderIntoDocument(<Question {...props} />);

    let link = TestUtils.findRenderedComponentWithType(doc, Link);

    expect(link).not.to.be.undefined;
    expect(link.props.to).to.equal('/');
  });
});
```
在這邊我們用 `react-addons-test-utils` 來 query component 裡面的東西並且加以驗証。
其中我們要測試的第二個行為："`component` 會 render 出一個指向 `/` 的 `Link`" 比較需要說明一下。
在這個測試裡面，**因為我們的測試對象是 `Question Component`, 所以我們並不在乎別的 component 內部是怎麼運作的。我們只在乎 `Question` component 和別的 component 之間的介面關係而已。** 以 `Question` 為例，在它的測試裡面我們並不想要直接 render 別的 component (`Link`)，原因是因為別的 component 在 render 的時候有可能會衍生出其他的問題，當這種狀況發生的時候，我們很難確定究竟是 `Question` 的問題還是其他 component 的問題。
這樣的狀況在 component 的測試時常發生，因為大多數的時候，我們會在一個 component 去 render 其他很多不同的 component。

面對這種問題我的解決方法是**在測試裡面，用 `__Rewire__` 把其他 component mock 掉 **，

```javascript
    Link = React.createClass({
      render() {
        return (<div>MOCK COMPONENT CLASS</div>)
      }
    });
    Container.__Rewire__('Link', Link);
```
這樣一來，`Question` 在測試裡面被 render 的時候，他看到的 `Link` 就會是我們提供的假的 component 而不是真正的 `Link` component 了。接著我們就可以進一步去測試 `Question` component 和 `Link` component 之間的介面關係：

```javascript
    let doc = TestUtils.renderIntoDocument(<Question {...props} />);
    let link = TestUtils.findRenderedComponentWithType(doc, Link);

    expect(link).not.to.be.undefined;
    expect(link.props.to).to.equal('/');
```

## 該如何面對 webpack
webpack 有很多很神妙的 loaders，可以讓我們在 javascript 裡面 `require` 各種不是 javascript 的東西，像是圖片、stylesheet 等等。在選用這樣的 loaders 的時候，當中會牽涉到一些取捨。
**選用 loader 的好處是，所有的東西都被打包在一起了；但它的壞處是，用到這些 loader 的 code 就只能在 webpack 的環境下執行了**。
如果說我們的 app 只會在瀏覽器上跑，那用各種 loader 基本上是沒什麼問題。**但如果同一份 code 也要在 server 上執行的話 (universal rendering)，那選用 loader 就代表在 server 上跑的 code 要用 webpack 分開打包，也代表著我們會需要兩份 webpack config。**
我自己的偏好是，如果要做 universal rendering, 我會避開這類的 loader, 讓 server 的 code 可以不用透過 webpack 來 build，對我來說這樣比較單純。

下面我提供的例子是一個只有在瀏覽器裡執行的例子(**不在**[這個範例](https://github.com/mz026/universal-redux-template)裡面)，其中用到了 url loader 來 require 圖片。

```javascript
let SignupModal = React.createClass({
  render() {
    let cmLogo = require('Icons/white/icon_codementor.png');

    ...
  }
})
```

這個 component 會在 render 的時候，去 require 一個圖片，然後在 nodejs 的環境下測試就壞掉惹。
我的解法是，**用一個 function 把 require image 的行為包起來，這樣一來我們就可以在測試的時候去 mock 這個 function。**

於是我們的 component 變成這樣：
```javascript
import requireImage from 'lib/requireImage';

let SignupModal = React.createClass({
  render() {
    let cmLogo = requireImage('Icons/white/icon_codementor.png');

    ...
  }
})
```

其中 `requireImage` 就只是單純地去 `require` 而已: `lib/requireImage.js`:

```javascript
export default function(path) {
  return require(path);
}
```

這樣一來，我們就可以在測試裡面 mock 掉 `requireImage` 了：

```javascript
describe('Component SignupModal', function(){
  let requireImage;
  beforeEach(function() {
    requireImage = function() {}
    SignupModal.__Rewire__('requireImage', requireImage);
  });

  it('can be rendered', function() {
    // now we can render here
  });
});
```

## 結論
測試是一件需要花力氣去做的事情，同時也需要很多練習。但是透過這個過程，我們可以更了解整份 code 的運作方式。
在大部份的時候，容易測試的 code 也會是有好的結構的 code。
當一個 project 變得越來越大，開發者越來越多，沒有測試的 code 到後來幾乎會被 bug 壓垮。而當我們越來越熟練之後，其實寫測試的時間絕對是遠小於 debug 的時間的。

Redux 的設計讓測試變得很單純，我覺得這也是它很精妙的地方之一。
好測試，不寫嗎？
