---
title: webpack 打包过程
date: 2019-07-17 23:56:56
tags:
---
本文参考`webpack`创始人 Tobias Koppers 的视频 [Webpack founder Tobias Koppers demos bundling live by hand](https://www.youtube.com/watch?v=UNMkLHzofQI), 以及开源项目[minipack](https://github.com/ronami/minipack)，梳理了`webpack`打包过程。
# 手动打包文件
## 文件目录
我们准备一个极简单的项目来进行打包，目录结构和内容如下：
```
+-- src
| +-- big.js
| +-- helloWorld.js
| +-- index.js
| +-- lazy.js
```
index.js
```javascript
import helloWorld from './helloWorld'
const node = document.createElement("div")
node.innerHTML = helloWorld + 'loading...'
import(/* webpackChunkName: "async" */ './lazy').then(({ default: lazy }) => {
node.innerHTML = helloWorld + lazy
})
document.body.appendChild(node)
```
helloWorld.js
```javascript
import big from './big'
const helloWorld = big('hello world!')
export default helloWorld
```
big.js
```javascript
export default (val) => {
return val && val.toUpperCase()
}
```
lazy.js
```javascript
import big from './big'
const lazy = big("lazy loaded!")
export default lazy
```
我们先来看下webpack打包之后的结果，省略了一些代码，但是大体可以看到，所有分散的文件最终变成一个立即执行函数，参数是文件（模块）队列数组。
```javascript
 /******/ (function(modules) {// webpackBootstrap
   /******/ // ...
 /******/ })({
/************************************************************************/
/***/ "./src/big.js":
/***/ (function(module, __webpack_exports__, __webpack_require__) {}),
/***/ "./src/index.js":
/***/ (function(module, __webpack_exports__, __webpack_require__) {
/***/ })

/******/ });

```
我们第一个目标是通过人工打包的方式生成这样一个立即执行函数，通过一定的串联逻辑，将所有的模块整合到一起。

## 模块划分
可以看到项目模块之间有这样的引用关系，入口文件引入了`helloWorld`和`lazy`，`helloWorld`和`lazy`引入了`big`

```
* src/index.js (ESM)
    #  ./helloWorld
    # (async) ./lazy
    -  src/helloWorld.js
    -  (async) src/lazy.js
* src/helloWorld.js (ESM)
    # ./big
    -  src/big.js
* src/big.js
* src/lazy.js (ESM)
    # ./big
    -  src/big.js
 ```
 打包之后将会有两个文件，一个主文件main.js，一个是动态引入的async.js。其中，main是async的父文件，main中有的模块，asycn可以不引入。main文件里面已经包含了`src/big.js`，这里进行优化，打包后的`async.js`不需要包含`src/big.js`
 如下图所示：
 ![](miniWebpack/5b4526ed.png)
{% asset_img 5b4526ed.png %}
 
 main.js
 ```
 - src/index.js
 - src/helloWorld.js
 - src/big.js
 ```
 async.js (parent:main)
 ```
 - src/lazy.js
 - src/big.js （ in parent）---delete
 ```
 现在划分一下模块，可以看到入口文件--`index.js`，我们将它`import`的文件直接串联当成第一个模块。这里只有引入一个模块`helloWorld`（lazy是打包进去`async.js`暂不考虑）。由此可以划分成三个模块，我们手动为每个模块赋予一个`id`（中括号中的数字）。
 ```
* [0]src/index.js (ESM) + 1modules
    #  ./helloWorld
    # (async) ./lazy
    -  src/helloWorld.js
    -  (async) src/lazy.js
    -  src/big.js
* [1]src/big.js
* [2]src/lazy.js (ESM)
    # ./big
    -  src/big.js
 ```
 ## import和export
 我们知道`webpack`把分散的代码通过`import`和`export`串成一个立即执行函数（IIFE），参数是模块对象数组。
 其中模块对象是一个这样的结构：
 ```js
 {
  [moduleId]: function() {
      // 模块代码
  }
}
 ```
现在来处理一下每个文件的`import`和`export` 。

对于每一个模块，要保证有独立的作用域，用一个`funtion`去包裹。并且传入两个参数，用来实现`import`和`export`的功能。
 index.js + 1 modules(hellowWorld.js)
 ```javascript
(function(__require__, exports) {
  let X = __require__(1)
  const helloWorld = X.default('hello world!')
  const node = document.createElement("div")
  node.innerHTML = helloWorld + 'loading...'
  // 先看普通的import
  // import(/* webpackChunkName: "async" */ './lazy').then(({ default: lazy }) => {
  //   node.innerHTML = helloWorld + lazy
  // })
  document.body.appendChild(node)
})
```
big.js
```javascript
(function(__require__, exports) {
  exports.default = (val) => {
    return val && val.toUpperCase()
  }
})
```
 ### 模拟import
 import的功能就是：
 1.执行目标模块的代码;
 2.导出目标模块的`export`内容给外部使用。
 如下`__require__`函数的实现，
 ```javascript
function __require__(id) {
    // 设置一个缓存，有的话直接返回
    if(cache[id]) return cache[id].exports
    
    var module = {
        exports: {}
    };
    // 1、执行当前模块的内容，这个modules[id]就是我们刚才对每个模块封装的那个方法
    modules[id](__require__, module.exports, module)
    cache[id] = module
    // 2、导出当前模块的export内容给外部使用
    return module.exports
}
```

runtime.js
```javascript
!(function(modules){
    function __require__ (id) {
        var module = {
            exports: []
        }
        modules[id](__require__, module.exports, module);
        return module.exports
    }
    __require__(0)
    })(
    {
        0: (function(__require__, exports) {
                let X = __require__(1)
                const helloWorld = X.default('hello world!')
                const node = document.createElement("div")
                node.innerHTML = helloWorld + 'loading...'
                // import(/* webpackChunkName: "async" */ './lazy').then(({ default: lazy })   => {
                // node.innerHTML = helloWorld + lazy
                // }
                document.body.appendChild(node)
              }),
        1: (function(__require__, exports) {
                exports.default = (val) => {
                return val && val.toUpperCase()
                }
            })
    }
)
```
在index.html上引入这个文件，打开就能看到结果了。至此，我们完成了最基本的手动打包流程。

### 懒加载模块
现在还剩下对`lazy.js`的打包，它是作为一个单独的文件，按需引入的。我们希望使用的时候是这样的，加载完模块，然后进行require：
index.js (bundled)
  ```javascript
// import(/* webpackChunkName: "async" */ './lazy').then(({ default: lazy }) => {
// node.innerHTML = helloWorld + lazy
// })
__require__.loadChunk(0)
    .then(__require__.bind(null, 3))
        .then(function(Y){
            node.innerHTML = helloWorld + Y.default
        })
```

请求一个文件地址，得到文件中的数据，这个过程用类似`jsonp`的方式来实现。
![](miniWebpack/2019-07-28-00-18-44.png)
{% asset_img 2019-07-28-00-18-44.png %}

首先是下载文件，这个过程是异步的，要用一个`promise`来封装。下载完成，还需要解析出数据才能执行下一步。所以，`promise`的回调函数`resolve`下载完成先放在一个全局变量`chunkResolves`当中，等解析出数据之后再调用它。

runtime.js
```javascript
// 每个模块下载(promise)完成对应的resolve
let chunkResolves = {};

__require__.loadChunk = function(chunkId) {
    return new Promise(resolve => {
        chunkResolves[chunkId] = resolve
        let script = document.createElement('script')
        script.src = 'src/' + {0: 'async'}[chunkId]+ '.js'
        document.head.appendChild(script)
    })
}
```
根据`jsonp`的原理，下载下来的模块对象需要用一个`callback`（这里是`requireJsonp`）包裹，变成一个可执行的脚本，下载完成之后在本地执行这个`callback`才能解析出模块对象。所以手动对异步的模块进行一个封装：
async.js
```javascript
window.requireJsonp( 0, {
    3: (function(__require__, exports) {
        let X = __require__(1)
        const lazy = X.default("lazy loaded!")
        exports.default = lazy
    })
})
```
并且我们应提前声明好`window.requireJsonp`这个回调函数。我们把下载得到的动态模块对象添加到立即执行函数参数的个模块对象，就回到了普通的模块打包的情况，这时候解析完成，执行`promise`的`resolve`，算是整个异步加载的过程结束。
runtime.js
```javascript
!(function(modules){
    function __require__ (id) {
     // ...
    }
    // 每个模块下载(promise)完成对应的resolve
   let chunkResolves = {};
   
    window.requireJsonp = function(chunkId, newModules) {
        for (const id in newModules) {
            modules[id] = newModules[id]
            chunkResolves[chunkId]();
        }
    }
    __require__(0)
    })({
        //...模块对象
    })
```
这样，我们就完成了人工打包一个项目的简单流程。接下来看要怎么用代码来实现自动打包。
# 自动打包
 先不看详细的细节，我们主要的步骤就是:
 ```js
// 解析模块
function createAsset(filename) {}
// 生成依赖图
function createGraph(entry){}
// 打包
function bundle(graph){}
 
const graph = createGraph('./src/index.js')
const result = bundle(graph)
 ```
 所依赖的工具
 * abylon：js解析器，将文本代码转化成AST（语法树）
 * babel-travse：遍历AST寻找依赖关系
 * babel-core的transformFromAst：将AST代码转化成浏览器所能识别的代码(ES5)

```js
const fs = require('fs');
const path = require('path');
const babylon = require('babylon'); // 将文件转化成AST
const traverse = require('babel-traverse').default; // 寻找依赖关系
const {transformFromAst} = require('babel-core'); // 将 AST 转化成 ES5
```
 主要就是把文本文件转化成语法树，拿到`import`和`export`知道模块之间的依赖关系，再把语法树转换成ES5。

可以了解一下语法树如下图所示，可以拿到每句代码对应的信息。
![](miniWebpack/2019-07-28-01-23-40.png)
{% asset_img 2019-07-28-01-23-40.png %}
