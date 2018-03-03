---
layout: post
title: 如何與 Webpack Environment Variables 成為好友
comments: true
tags:
  - webpack
  - "environment-variable"
  - react
  - javascript
---

在 React 的開發工具當中， [Webpack](https://webpack.github.io/) 可以說是相當<s>詭譎</s>神妙的一員。
他在某些方面取代了很多本來用 gulp/grunt 做的事情，像是 pre-process, post-process javascript/css 等，但同時也因為它的設定比較複雜，有時候蠻令人頭大的。

我想在這篇文章分享我自己在使用 webpack 的時候，遇到關於 environment variables (環境變數) 的問題和解法。

## 情境設定：
在開發的時候，我們常常會把一些設定參數、thrid-party service keys 等等的東西放在環境變數裡面 (而不是直接 commit 進去 repository)。像是 API\_HOST 的 url，或者是 AWS 的 access key 等等。
所以在我們的程式裡面可能會有以下這樣的做法：

```javascript
// app.js
var config = require('./config');
var agent = require('superagent');

agent.get(config.apiHost + '/the-api-path');
```

而其中的 config 檔可能長這樣：

```javascript
// config.js
module.exports = {
  apiHost: process.env['API_HOST'] || 'http://localhost:3000'
}
```

如此一來，我們就可以把不同環境的 API\_HOST 設定在 environment variable (`$API_HOST`)裡面，並且給定一個預設的值 (`http://localhost:3000`)。

但是在把上面的 code 用 webpack build 起來之後，我們發現事情並<s>不是憨人所想的那麼簡單</s>沒有那麼單純。

## Webpack v.s. Environment Variables

為了方便說明，我們可以先暫時把上面的 `app.js` 改成這樣：

```javascript
var config = require('./config');
console.log(config.apiHost);
```

這樣我們就可以直接執行 build 好的 code 然後把結果 log 出來。

### 直接 build 下去

我們如果把上面的 code 直接 build 起來：

```
$ webpack app.js --output-filename bundle.js
```

然後接著執行 `bundle.js`:

```
$ API_HOST=http://production-api.com node bundle.js
```

這邊我們預期要得到的結果應該是 `http://production-api.com`
但實際上的結果卻是 `http://localhost:3000`

**也就是說，`bundle.js` 在執行的時候，完全不管我們 pass 進去的 environment variable 阿！**

### 在 build 的時候 pass 環境變數

接著我們試著在 build 的當下就 pass environment variable 進去。其實這樣的做法比較接近實際上 deployment 的狀況 (像是在 Heroku 上面設定好相對應的 environment variables, 然後在 server 上面把 code build 起來)

```
$ API_HOST=http://production-api.com webpack app.js --output-filename bundle.js
```

然後同樣執行 build 好的 `bundle.js`

```
$ API_HOST=http://production-api.com node bundle.js
```

什麼？！！結果還是一樣阿靠！

執行的結果還是一樣是 `http://localhost:3000` 而不是我們想要的 `http://production-api.com`！

在直接把 `bundle.js` 打開來看之後，我們發現他裡面寫著：

```javascript
	process.env = {};
```

難怪 pass 什麼東西進去都沒用阿。

## DefinePlugin
在經過一番 google 之後，發現原來可以用 [webpack 的 DefinePlugin](https://webpack.github.io/docs/list-of-plugins.html#defineplugin) 來解決這個問題。我們新加一個檔案 `webpack.config.js`：

```javascript
// webpack.config.js
var webpack = require('webpack');
var envToBeInjected = {
  API_HOST: process.env.API_HOST
};

module.exports = {
  plugins: [
    new webpack.DefinePlugin({
      'process.env': JSON.stringify(envToBeInjected)
    })
  ]
};
```

然後再執行上面的 build script:
```
$ API_HOST=http://production-api.com webpack app.js --output-filename bundle.js
```

然後再試一下：
```
$ node bundle.js
```

耶！終於出現夢寐以求的 `http://production-api.com` 了。

## How It works
上面我們在 `webpack.config.js` 裡面，用 `DefinePlugin` 把 `process.env` 這個變數用
```javascript
{ API_HOST: process.env.API_HOST }
```
代換進去。

`webpack.config.js` 是在 build 時候被執行，這就是為什麼我們要在 build 的時候 pass `API_HOST=http://production-api.com` 的原因。而在 `webpack.config.js` 裡特別把 `API_HOST` 抽出來而不是直接 pass 整個 `process.env` 的原因是因為，在 server 上面的 environment variables 裡面可能還有其他的東西像是其他 service 要用到的 key 等等。這樣東西還是不要一起 build 進去比較妥當 XD

而 webpack build 好的 `bundle.js` 是直接把 build 的當下的 environment variable 寫進去，這也是為什麼我們在最後執行 `bundle.js` 的時候，不需要在另外 pass `API_HOST` 的原因。
