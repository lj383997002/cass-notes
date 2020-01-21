# Webpack 打包文件分析

## Webpack 实现前端模块化

在webpack打包后的代码中，每个模块都有一个模块对象，其数据结构遵循 _**commonjs**_ 规范。

webpack 实现了 \_\_webpack\_require\_\_ 方法，相当于 Node 中的 require。

### 1. 简单打包文件分析

入口文件比较简单，没有动态导入等操作。

```javascript
// webpack.config.js
const path = require('path')
module.exports = {
  entry: path.resolve(__dirname, '../src/main.js'),
  output: {
    path: path.join(__dirname, '../build'),
    filename: 'bundle.js',
  },
}

// main.js
import { render } from './test.js'
render()
export default {
  a: 1,
}

// test.js
const render = () => {
  console.log('test module: ', test)
}
export { render }
```

根据上述代码，执行 **webpack --mode=development** 命令后，可得出下述 bundle.js

```javascript
 (function(modules) {
  // 缓存的模块
 	var installedModules = {};

   /**
    * // 模拟实现 node 中的 require 方法
    * @param {string} moduleId 模块ID
    * @returns {any} 模块导出内容
    */
 	function __webpack_require__(moduleId) {
    // 取出缓存中对应的模块, 如果有, 直接返回. 反之, 加载模块
 		if(installedModules[moduleId]) {
 			return installedModules[moduleId].exports;
 		}
     
    //  创建新的模块, 并且添加到缓存中
 		var module = installedModules[moduleId] = {
      // 模块ID
      i: moduleId,
      //  模块加载状态
      l: false,
      //  模块导出内容
 			exports: {}
 		};

    // 执行模块对应的封装方法, webpack会将每个模块用一个function包裹
 		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);

 		// 设置模块加载状态, true 为已加载
 		module.l = true;

 		// 返回模块导出内容
 		return module.exports;
 	}

  // 将所有需要加载的模块添加到 require 中, require方法会传递到子模块中, 递归执行
 	__webpack_require__.m = modules;

 	// 同上, 将所有缓存模块添加到 require 中
 	__webpack_require__.c = installedModules;

 	// 定义 __esModule es6导出方式
 	__webpack_require__.r = function(exports) {
 		if(typeof Symbol !== 'undefined' && Symbol.toStringTag) {
 			Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' });
 		}
 		Object.defineProperty(exports, '__esModule', { value: true });
 	};

 	// Object.prototype.hasOwnProperty.call
 	__webpack_require__.o = function(object, property) { return Object.prototype.hasOwnProperty.call(object, property); };

  // output publicPath
 	__webpack_require__.p = "";

  // 执行入口文件对应的模块
 	return __webpack_require__(__webpack_require__.s = "./src/main.js");
 })
 ({
  "./src/main.js":
  (function(module, __webpack_exports__, __webpack_require__) {
		"use strict";
		// 执行模块中的代码, 模块内部加载了 __webpack_require__('./test.js')
		// 随后加载 test.js 中的代码, 根据下述打包后的代码, 返回 { render }
		// 加载完后赋值 _test_js__WEBPACK_IMPORTED_MODULE_0__ = { render }, 并且调用
    eval("__webpack_require__.r(__webpack_exports__);\n/* harmony import */ var _test_js__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(/*! ./test.js */ \"./src/test.js\");\n\r\n\r\nObject(_test_js__WEBPACK_IMPORTED_MODULE_0__[\"render\"])()\r\n\r\n/* harmony default export */ __webpack_exports__[\"default\"] = ({\r\n  a: 1,\r\n});\r\n\n\n//# sourceURL=webpack:///./src/main.js?");
  }),
  "./src/test.js":
  (function(module, __webpack_exports__, __webpack_require__) {
    "use strict";
    eval("__webpack_require__.r(__webpack_exports__);\nconst render = () => {\r\n  console.log(1111111)\r\n}\r\n\r\n/* harmony default export */ __webpack_exports__[\"default\"] = (render);\r\n\n\n//# sourceURL=webpack:///./src/test.js?");
  })
 });
```

### 2. 添加动态导入（import\(\)）后的包文件分析

入口文件也比较简单，将 test.js 改成动态import就行

```text
// main.js
import('./test.js').then(({ render }) => {
  render()
})
export default {
  a: 1,
}
```

根据上述代码，执行 **webpack --mode=development** 命令后，可得出下述 bundle.js

```text
(function (modules) {
  // __webpack_require__.e 执行完毕后, 加载异步模块时, 会调用 window['webpackJsonp'].push
  // 相当于调用了 webpackJsonpCallback
  function webpackJsonpCallback(data) {};
  
  // 缓存模块
  var installedModules = {};

  // 缓存动态加载的 chunk 块
  // undefined = chunk not loaded, null = chunk preloaded/prefetched
  // Promise = chunk loading, 0 = chunk loaded
  var installedChunks = {
    "main": 0
  };

  // 返回动态加载 chunk 的 URL
  function jsonpScriptSrc(chunkId) {
    return __webpack_require__.p + "" + chunkId + ".bundle.js"
  }

  // webpack 模拟 require 方法
  function __webpack_require__(moduleId) {}

  // 根据模块ID, 异步加载模块(通过jsonp加载js)
  __webpack_require__.e = function requireEnsure(chunkId) {};

  // 将所有需要加载的模块添加到 require 中, require方法会传递到子模块中, 递归执行
  __webpack_require__.m = modules;

  // 同上, 将所有缓存模块添加到 require 中
  __webpack_require__.c = installedModules;

  // define __esModule on exports
  __webpack_require__.r = function (exports) {
    if (typeof Symbol !== 'undefined' && Symbol.toStringTag) {
      Object.defineProperty(exports, Symbol.toStringTag, {
        value: 'Module'
      });
    }
    Object.defineProperty(exports, '__esModule', {
      value: true
    });
  };

  // Object.prototype.hasOwnProperty.call
  __webpack_require__.o = function (object, property) {
    return Object.prototype.hasOwnProperty.call(object, property);
  };

  // output publicPath
  __webpack_require__.p = "";

  // 异步加载模块失败时处理的回调
  __webpack_require__.oe = function (err) {
    console.error(err);
    throw err;
  };

  // 绑定 webpackJsonp 的 push 方法
  var jsonpArray = window["webpackJsonp"] = window["webpackJsonp"] || [];
  // 绑定 jsonpArray 的 push 方法, 并且保存下来
  var oldJsonpFunction = jsonpArray.push.bind(jsonpArray);
  // 重置 push 方法, 调用 jsonpArray.push / window["webpackJsonp"].push 时, 会调用 webpackJsonpCallback
  jsonpArray.push = webpackJsonpCallback;
  jsonpArray = jsonpArray.slice();
  // 这里首次加载页面时, 数据是空
  for (var i = 0; i < jsonpArray.length; i++) webpackJsonpCallback(jsonpArray[i]);
  var parentJsonpFunction = oldJsonpFunction;

  // 执行入口文件对应的模块
  return __webpack_require__(__webpack_require__.s = "./src/main.js");
})
({
  "./src/main.js":
    (function (module, __webpack_exports__, __webpack_require__) {
      "use strict";
      // 执行当前入口文件代码, 并且调用 __webpack_require__.e(moduleId) 加载对应的异步模块(import())
      eval("__webpack_require__.r(__webpack_exports__);\n__webpack_require__.e(/*! import() */ 0).then(__webpack_require__.bind(null, /*! ./test.js */ \"./src/test.js\")).then(({ render }) => {\r\n  render()\r\n})\r\n/* harmony default export */ __webpack_exports__[\"default\"] = ({\r\n  a: 1,\r\n});\n\n//# sourceURL=webpack:///./src/main.js?");
    })
});
```

根据上述代码，执行 _**\_\_webpack\_require\_\_\('./src/main.js'\)**_ 时，会执行入口文件的代码， 由于入口文件中动态加载了 _**test.js**_ ，webpack默认会对动态加载的代码进行代码分割，所以会执行 \_\_webpack\_require\_\_.e。

```text
// function __webpack_require__.e 
__webpack_require__.e = function requireEnsure(chunkId) {
  var promises = [];

  // 通过 jsonp 加载异步模块

  // 取出缓存中的模块
  var installedChunkData = installedChunks[chunkId];
   // 0: 模块已经加载, Promise: 正在加载
   // undefined: 没有加载, null: webpack chunk prefetched/preloaded
  if (installedChunkData !== 0) {
    // 如果有数据, 则意味着数据为 [resolve, reject, Promise]
    if (installedChunkData) {
      // 将正在加载的模块添加到队列中
      promises.push(installedChunkData[2]);
    } else {
      // 如果没有加载, 则加载模块

      // 创建 Promise, 添加到队列中, 并且将正在加载的数据添加到缓存对象 installedChunks 中
      var promise = new Promise(function (resolve, reject) {
        installedChunkData = installedChunks[chunkId] = [resolve, reject];
      });
      promises.push(installedChunkData[2] = promise);

      // jsonp 原理, 动态创建 script 标签, script天然支持跨域
      var script = document.createElement('script');
      var onScriptComplete;
      // 设置 字符集
      script.charset = 'utf-8';
      // 设置 script 超时时间
      script.timeout = 120;
      if (__webpack_require__.nc) {
        script.setAttribute("nonce", __webpack_require__.nc);
      }
      // 设置 src
      script.src = jsonpScriptSrc(chunkId);

      // create error before stack unwound to get useful stacktrace later
      var error = new Error();
      // 判断资源是否加载异常
      onScriptComplete = function (event) {
        // avoid mem leaks in IE.
        script.onerror = script.onload = null;
        clearTimeout(timeout);
        var chunk = installedChunks[chunkId];
        // 资源加载完毕后, 如果状态不为0, 
        if (chunk !== 0) {
          if (chunk) {
            var errorType = event && (event.type === 'load' ? 'missing' : event.type);
            var realSrc = event && event.target && event.target.src;
            error.message = 'Loading chunk ' + chunkId + ' failed.\n(' + errorType + ': ' + realSrc + ')';
            error.name = 'ChunkLoadError';
            error.type = errorType;
            error.request = realSrc;
            chunk[1](error);
          }
          installedChunks[chunkId] = undefined;
        }
      };
      // 设置请求超时时间
      var timeout = setTimeout(function () {
        onScriptComplete({
          type: 'timeout',
          target: script
        });
      }, 120000);
      script.onerror = script.onload = onScriptComplete;
      document.head.appendChild(script);
    }
  }
  // 返回加载中的所有 Promise
  return Promise.all(promises);
}
```

上述 \_\_webpack\_require\_\_.e 执行完毕后，**如果jsonp请求js文件没有异常，则加载异步模块代码。**

```text
// test.bundle.js
// 根据前面说的 webpackJsonp.push 方法绑定, 执行push时, 会调用 webpackJsonpCallback
(window["webpackJsonp"] = window["webpackJsonp"] || []).push([[0],{
  "./src/test.js":
  (function(module, __webpack_exports__, __webpack_require__) {
    "use strict";
    eval("__webpack_require__.r(__webpack_exports__);\n/* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, \"render\", function() { return render; });\nconst render = () => {\r\n  console.log(1111111)\r\n}\r\n\r\n\r\n\n\n//# sourceURL=webpack:///./src/test.js?");
  })
}]);
```

异步加载模块执行时，webpackJsonpCallback 会调用。

```text
// webpackJsonpCallback
/**
 * data = [chunkId, module]
 * chunkId: 分割出来的代码块的ID
 * module: key-value对象, key为模块路径, value为模块通过 function 包装后的整体代码
 */
function webpackJsonpCallback(data) {
  var chunkIds = data[0];
  var moreModules = data[1];

  var moduleId, chunkId, i = 0,
    resolves = [];
  // 将所有异步加载模块的状态改成加载成功, 并且将 resolve 方法添加到队列中
  for (; i < chunkIds.length; i++) {
    chunkId = chunkIds[i];
    if (Object.prototype.hasOwnProperty.call(installedChunks, chunkId) && installedChunks[chunkId]) {
      resolves.push(installedChunks[chunkId][0]);
    }
    installedChunks[chunkId] = 0;
  }
  // 将异步加载模块的所有依赖模块添加到缓存中
  for (moduleId in moreModules) {
    if (Object.prototype.hasOwnProperty.call(moreModules, moduleId)) {
      modules[moduleId] = moreModules[moduleId];
    }
  }
  // 将数据存入到 webpackJsonp 中
  if (parentJsonpFunction) parentJsonpFunction(data);

  // 执行所有 promise resolve 方法
  // 执行完毕后, promise 状态结束, 继续执行 main.js 中的 then 方法
  while (resolves.length) {
    resolves.shift()();
  }
}
```

webpackJsonpCallback 执行完毕后，调用栈回到了 main.js 中，此时才会去真正执行 test.js 中的代码。在**webpack\_require**.e\(0\).then\(**webpack\_require**.bind\(null, ‘./src/test.js’\)\) 中，可以看到通过**webpack\_require**去加载 test.js, 之后的流程与初始化 main.js 是一样的。至此，整个动态加载异步模块的包文件分析完毕。

### 3. 配置 webpack.optimization.runtime 

上述配置项配置后，会将依赖文件与包文件分离，生成 runtime 运行时文件。 此时打包后的文件与上述两种方法大致相同，主要多了 **checkDeferredModules** 方法，此方法比较简单。

```javascript
function checkDeferredModules() {
  // 取出加载文件列表, 并且判断需要执行的js的依赖项是否加载完毕
  // 如果没有则跳过, 等到依赖项加载完成后再执行
  // eg: deferredModule = ['./src/main.js', 'runtime~main'], 如果需要加载 main.js, 此时 runtime~main 依赖项需要加载完成
	var result;
	for(var i = 0; i < deferredModules.length; i++) {
		var deferredModule = deferredModules[i];
		var fulfilled = true;
		for(var j = 1; j < deferredModule.length; j++) {
			var depId = deferredModule[j];
			if(installedChunks[depId] !== 0) fulfilled = false;
		}
		if(fulfilled) {
			deferredModules.splice(i--, 1);
			result = __webpack_require__(__webpack_require__.s = deferredModule[0]);
		}
	}
	return result;
}
```

综合上述三种方案，webpack基础打包文件分析已经完成，如果需要添加其他配置项，添加后分析即可。

