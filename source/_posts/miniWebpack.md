---
title: miniWebpack
date: 2019-07-17 23:56:56
tags:
---
# 手动打包文件
## 模块划分
我们拥有当前的目录结构：
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
我们可以看到模块间的引用关系

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
 
 现在来划分一下模块，可以看到入口文件--`index.js`，它比较特殊，没有export，我们将它`import`的文件直接串联当成第一个模块。这里只有引入一个模块`helloWorld`（lazy是打包进去`async.js`暂不考虑）。由此可以划分成三个模块。
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
 webpack最主要的功能就是打包，把分散的代码通过import和export串成一个立即执行函数（IIFE）。
 ```javascript
!(function(modules){
  // 模块串联逻辑
  })({
  // 所有的模块对象
  })
```
模块对象是一个这样的结构
```javascript
{
  [moduleId]: code..
}
```
我们在上文整理的三个模块，对应这个模块对象的三个键值对。现在继续用手动的方式先来处理一下这个模块对象。
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
 impoor的功能就是：
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
将模块串联逻辑和模块id键值对填充进刚才的IIFE
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
在index.html上引入这个文件，正式打开就能看到结果了。至此，我们完成了最基本的手动打包流程。

### 懒加载模块
现在还剩下对`lazy.js`的打包，它是作为一个单独的文件，按需引入的。请求一个文件地址，返回一个可执行的函数，这个形式就类似于jsonp。`lazy代码-id`的键值对就是我们需要的json数据。我们在IIFE函数的`__require__`上添加一个方法`loadchunk`执行文件的下载。文件的下载是异步的，要封装一个promise，以供加载完成之后进行下一步的操作。
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
现在先来手动封装一下`lazy.js`打包之后的`async.js`文件。请求`async.js`文件执行`window.requireJsonp`方法拿到数据包。
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
对于`window.requireJsonp`方法，在`async.js`我们传入了`chunkId`和我们需要的模块数据包。现在要在下载完成解析出来。
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
  使用`loadChunk`模拟动态`import`。
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
 
 
