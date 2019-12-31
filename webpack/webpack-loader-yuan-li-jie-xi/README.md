# Webpack Loader 原理解析

### loader runner

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

#### runLoaders

```text
function runLoaders(options, callback) {
  // 解构 options 中的参数
	var resource = options.resource || "";
	var loaders = options.loaders || [];
	var loaderContext = options.context || {};
	var readResource = options.readResource || readFile;

	var splittedResource = resource && splitQuery(resource);
	var resourcePath = splittedResource ? splittedResource[0] : undefined;
	var resourceQuery = splittedResource ? splittedResource[1] : undefined;
	var contextDirectory = resourcePath ? dirname(resourcePath) : null;

	// execution state
	var requestCacheable = true;
	var fileDependencies = [];
	var contextDependencies = [];
}
```

