# Webpack Loader 原理解析

## loader runner

根据官网描述：loader 为模块加载器，处理对应的资源。在webpack构建过程中，解析文件依赖时，会执行 loader runner 中的 runLoaders 方法。本篇笔记在解析loader执行过程后，会分析几个loader源码。

`node-nightly --inspect --inspect-brk ./node_modules/webpack/bin/webpack.js  --config ./config/webpack.config.js --mode=development`

执行上述命令，将会在webpack库 package.json 中 main 配置项对应的 lib/webpack.js 的第一行打上断点。手动在 loader-runner 库中植入debugger，跳过断点即可看到执行过程。

```text
// ./config/webpack.config.js
const path = require('path')

module.exports = {
  entry: path.resolve(__dirname, '../src/main.js'),
  output: {
    path: path.join(__dirname, '../build'),
    filename: 'bundle.js'
  },
  module: {
    rules: [
      {
        test: /\.css$/,
        exclude: /node_modules/,
        use: [
          'style-loader',
          'css-loader',
        ],
      },
    ],
  },
}

// webpack.js -> NormalModule.js
doBuild(options, compilation, resolver, fs, callback) {
  // 创建上下文对象, loader function 中的 this
  const loaderContext = this.createLoaderContext(
  	resolver,
  	options,
  	compilation,
  	fs
  );

  // 执行 loader
  runLoaders(
  	{
      // 当前文件的绝对路径
  		resource: this.resource,
      // webpack.config.js 中的所有 loader
  		loaders: this.loaders,
      // loader上下文
  		context: loaderContext,
      // 读取文件方法
  		readResource: fs.readFile.bind(fs)
  	},
  	(err, result) => {}
  );
}
```

### runLoaders\(version: 2.4.0\)

模块加载器的入口执行函数

```text
function runLoaders(options, callback) {
  // 解构 options 中的参数
	// 当前文件的绝对路径
	var resource = options.resource || "";
	// 处理当前资源对应的 loader
	var loaders = options.loaders || [];
	// loader 上下文
	var loaderContext = options.context || {};
	// 系统读取文件的方法
	var readResource = options.readResource || readFile;

	// 读取路径中的 query
	var splittedResource = resource && splitQuery(resource);
	var resourcePath = splittedResource ? splittedResource[0] : undefined;
	var resourceQuery = splittedResource ? splittedResource[1] : undefined;
	// 当前资源所在目录的绝对路径
	var contextDirectory = resourcePath ? dirname(resourcePath) : null;

	// 执行状态
	var requestCacheable = true;
	// 文件依赖
	var fileDependencies = [];
	// 上下文依赖
	var contextDependencies = [];
	
	// 将 loader 处理成对应的loader对象, 包含 pitch, data 等属性
	loaders = loaders.map(createLoaderObject);

	loaderContext.context = contextDirectory;
	// 记录当前正在执行的 loader 对应的位置
	loaderContext.loaderIndex = 0;
	loaderContext.loaders = loaders;
	loaderContext.resourcePath = resourcePath;
	loaderContext.resourceQuery = resourceQuery;
	
	// 编写异步 loader 时, 通过 this.async 返回一个 callback, 
	// 调用这个 callback 结束执行当前loader, 并将结果返回给之后的loader
	loaderContext.async = null;
	// 通过 this.callback 结束当前loader
	loaderContext.callback = null;
	loaderContext.cacheable = function cacheable(flag) {
		if(flag === false) {
			requestCacheable = false;
		}
	};
	// 挂载添加文件依赖的方法
	loaderContext.dependency = loaderContext.addDependency = function addDependency(file) {
		fileDependencies.push(file);
	};
	// 挂载添加上下文的方法
	loaderContext.addContextDependency = function addContextDependency(context) {
		contextDependencies.push(context);
	};
	// 挂载获取文件依赖的方法
	loaderContext.getDependencies = function getDependencies() {
		return fileDependencies.slice();
	};
	// 挂载获取上下文的方法
	loaderContext.getContextDependencies = function getContextDependencies() {
		return contextDependencies.slice();
	};
	// 清除所有依赖
	loaderContext.clearDependencies = function clearDependencies() {
		fileDependencies.length = 0;
		contextDependencies.length = 0;
		requestCacheable = true;
	};
	// 设置 resource 属性
	// 读取时, 值为当前资源路径及资源参数
	// 赋值时, 重置资源路径及资源参数
	Object.defineProperty(loaderContext, "resource", {
		enumerable: true,
		get: function() {
			if(loaderContext.resourcePath === undefined)
				return undefined;
			return loaderContext.resourcePath + loaderContext.resourceQuery;
		},
		set: function(value) {
			var splittedResource = value && splitQuery(value);
			loaderContext.resourcePath = splittedResource ? splittedResource[0] : undefined;
			loaderContext.resourceQuery = splittedResource ? splittedResource[1] : undefined;
		}
	});
	// 设置 request 属性, 将loader模块的入口文件与当前资源路径通过!内联
	// eg: style-loader!css-loader!main.css
	Object.defineProperty(loaderContext, "request", {
		enumerable: true,
		get: function() {
			return loaderContext.loaders.map(function(o) {
				return o.request;
			}).concat(loaderContext.resource || "").join("!");
		}
	});
	// 获取剩余处理资源的所有loader, 并且通过!内联当前资源
	// eg: 当前处理loader为 style-loader, 所以 remainingRequest: css-loader!main-css
	Object.defineProperty(loaderContext, "remainingRequest", {
		enumerable: true,
		get: function() {
			if(loaderContext.loaderIndex >= loaderContext.loaders.length - 1 && !loaderContext.resource)
				return "";
			return loaderContext.loaders.slice(loaderContext.loaderIndex + 1).map(function(o) {
				return o.request;
			}).concat(loaderContext.resource || "").join("!");
		}
	});
	// 获取正在处理资源的loader, 并通过!内联当前资源
	Object.defineProperty(loaderContext, "currentRequest", {
		enumerable: true,
		get: function() {
			return loaderContext.loaders.slice(loaderContext.loaderIndex).map(function(o) {
				return o.request;
			}).concat(loaderContext.resource || "").join("!");
		}
	});
	// 获取 preLoaders, 并通过!内联当前资源
	Object.defineProperty(loaderContext, "previousRequest", {
		enumerable: true,
		get: function() {
			return loaderContext.loaders.slice(0, loaderContext.loaderIndex).map(function(o) {
				return o.request;
			}).join("!");
		}
	});
	// 获取当前loader的options中的query参数
	Object.defineProperty(loaderContext, "query", {
		enumerable: true,
		get: function() {
			var entry = loaderContext.loaders[loaderContext.loaderIndex];
			return entry.options && typeof entry.options === "object" ? entry.options : entry.query;
		}
	});
	// 获取当前loader的data参数
	Object.defineProperty(loaderContext, "data", {
		enumerable: true,
		get: function() {
			return loaderContext.loaders[loaderContext.loaderIndex].data;
		}
	});
	
	iteratePitchingLoaders(...)
}
```

上述源码为 runLoaders 的初始化参数，参数初始化后，会执行 iteratePitchingLoaders。

根据官网在 Pitching loader 中提到：**loader 总是从右往左，从下往上执行。但是在有些情况下，loader 只关心 request 后的元数据，并忽略前一个loader的处理结果。在实际执行loader之前，会先从左到右调用loader的 pitch 方法。**

```text
use: [
  'a-loader',
  'b-loader',
  'c-loader'
]

|- a-loader `pitch`
  |- b-loader `pitch`
    |- c-loader `pitch`
      |- requested module is picked up as a dependency
    |- c-loader normal execution
  |- b-loader normal execution
|- a-loader normal execution
```

在上述loader的处理过程中，如果loader的pitch方法没有返回结果，则会按正常流程处理。如果其中一个pitch有返回结果，则会跳过剩下的loader，执行之前的 normalLoaders。

### iteratePitchingLoaders

处理loader时，会先从左往右执行所有loader的pitch方法

```text
function iteratePitchingLoaders(
  // 初始化参数, { buffer: null, readResource: fs.readFile }
  options,
  // loader 上下文
  loaderContext,
  callback,
) {
  /**
	 * 每执行一个 pitch, loaderIndex 会加 1
	 * 当所有 loader pitch 方法执行完毕后, 并且没有返回结果时, 处理当前资源
	 * 并回过头来从右往左执行所有 loaders
	 */
	if(loaderContext.loaderIndex >= loaderContext.loaders.length)
		return processResource(options, loaderContext, callback);

	// 获取当前正在处理的 loader
	var currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex];

	// 如果当前 loader 的 pitch 方法已经执行过, 则++, 继续往下执行
	if(currentLoaderObject.pitchExecuted) {
		loaderContext.loaderIndex++;
		return iteratePitchingLoaders(options, loaderContext, callback);
	}

	// 读取当前loader包文件, 并且挂载 normal, pitch, raw等参数
	// normal: loader 的导出函数
	// pitch: loader 的pitch函数
	loadLoader(currentLoaderObject, function(err) {
		if(err) {
			loaderContext.cacheable(false);
			return callback(err);
		}
		// 获取 loader 的 pitch, 并且设置执行状态
		var fn = currentLoaderObject.pitch;
		currentLoaderObject.pitchExecuted = true;
		// 如果没有 pitch, 则继续往下执行, 处理之后的loader pitch 方法, 直到最后一个处理完毕
		if(!fn) return iteratePitchingLoaders(options, loaderContext, callback);
		
		// 执行同步或者异步方法, 在这里会执行 pitch 方法
		runSyncOrAsync(fn, ...)
	});
}
```

### runSyncOrAsync

执行同步或者异步的公共方法

```text
function runSyncOrAsync(
  // loader pitch
  fn,
  // loader 上下文
  context,
  args,
  callback,
) {
  // 标识当前执行过程是同步还是异步, 默认同步
	var isSync = true;
	// 是否执行完毕
	var isDone = false;
	var isError = false; // internal error
	var reportedError = false;
	// 在写异步的loader时, 通过 this.async 获取 callback
	// 异步过程处理结束后, 执行 callback, 将结果传给剩余loader
	context.async = function async() {
		if(isDone) {
			if(reportedError) return; // ignore
			throw new Error("async(): The callback was already called.");
		}
		isSync = false;
		return innerCallback;
	};
	var innerCallback = context.callback = function() {
		if(isDone) {
			if(reportedError) return; // ignore
			throw new Error("callback(): The callback was already called.");
		}
		// 设置执行状态
		isDone = true;
		isSync = false;
		try {
			// 执行传入的 callback 方法, 并且将参数传入, 如果有返回值, 则跳过剩下的loader
			callback.apply(null, arguments);
		} catch(e) {
			isError = true;
			throw e;
		}
	};
	
	// 执行 pitch 方法, 并且获取返回值
	var result = (function LOADER_EXECUTION() {
		return fn.apply(context, args);
	}());
	// 同步执行的 loader
	if(isSync) {
		isDone = true;
		// 如果当前 pitch 没有返回值, 则继续往下执行
		if(result === undefined)
			return callback();
		// 返回值为 promise 的情况
		if(result && typeof result === "object" && typeof result.then === "function") {
			return result.then(function(r) {
				callback(null, r);
			}, callback);
		}
		// 如果 pitch 有返回值, 则跳过剩下的loader
		return callback(null, result);
	}
}
```

#### runSyncOrAsync Callback

```text
function(err) {
	if(err) return callback(err);
	// 获取处理 pitch 后的返回值
	var args = Array.prototype.slice.call(arguments, 1);
	// 如果有返回值, 则跳过剩下的 loader, 掉头执行 loader 的 normal 函数
	if(args.length > 0) {
		loaderContext.loaderIndex--;
		iterateNormalLoaders(options, loaderContext, args, callback);
	} else {
	// 如果没有返回值则继续往下执行 loader pitch
		iteratePitchingLoaders(options, loaderContext, callback);
	}
}
```

### iterateNormalLoaders

```text
function iterateNormalLoaders(
	
) {
	// loader 处理完毕, 退出执行
	// eg: style-loader!css-loader!main.css 处理style-loader时, 由于 pitch 有返回值
	// 跳过剩余的loader, 掉头执行 normal, 此时 loaderIndex -- < 0
	if(loaderContext.loaderIndex < 0)
		return callback(null, args);

	// 获取正在执行的loader
	var currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex];

	// 如果当前loader已经执行过, 继续往下执行
	if(currentLoaderObject.normalExecuted) {
		loaderContext.loaderIndex--;
		return iterateNormalLoaders(options, loaderContext, args, callback);
	}

	// 设置执行状态, 获取 normal 方法
	var fn = currentLoaderObject.normal;
	currentLoaderObject.normalExecuted = true;
	// 如果没有 normal, 则往下执行
	if(!fn) {
		return iterateNormalLoaders(options, loaderContext, args, callback);
	}
	
	// 将参数转成 buffer
	convertArgs(args, currentLoaderObject.raw);

	// 执行函数
	runSyncOrAsync(fn, loaderContext, args, function(err) {
		if(err) return callback(err);

		var args = Array.prototype.slice.call(arguments, 1);
		// 将上一个loader的执行结果传入, 递归执行, 直到所有loader处理完成
		iterateNormalLoaders(options, loaderContext, args, callback);
	});
}
```



