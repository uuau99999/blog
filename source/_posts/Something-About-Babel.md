---
title: Something About Babel
date: 2020-02-29 17:53:39
tags: Babel
---

## 关于 Babel 全家桶的理解

最近在研究公司一些公用库打包问题，尤其是关于如何减小包的体积思考了很久。加上自己之前一直对 babel 打包全家桶不是特别理解，故结合了掘金上面的几篇文章，在这里自己总结一下。

### 首先谈谈 babel 各全家桶的作用

- @babel/core  
  babel 的核心库，在 v7 以前版本叫 babel-core(其他的库也类似)，内置提供了 babel 的编译方法，如 transformFile, transformFileSync，调用后可得到根据特定 babel 配置的转译后代码，ast 等。

- @babel/cli  
  babel 的命令行工具，提供了在命令行中执行 babel 转译方法的指令。如
  ```
  npx babel script.js --out-file script-compiled.js
  ```
- babel-loader  
   提供一个在 webpack 中进行 babel 编译的 loader，如在 webpack.config.js 中配置：

  ```js
  {
  	rules: [
  		// the 'transform-runtime' plugin tells Babel to
  		// require the runtime instead of inlining it.
  		{
  			test: /\.m?js$/,
  			exclude: /(node_modules|bower_components)/,
  			use: {
  				loader: 'babel-loader',
  				options: {
  					presets: ['@babel/preset-env'],
  					plugins: ['@babel/plugin-transform-runtime']
  				}
  			}
  		}
  	];
  }
  ```

- @babel/polyfill  
  我们知道 babel 默认只会转换语法规范，不会在代码中注入新特性中的 API。如在 es6 中的 Object.assign 方法，babel 默认是不做任何处理的。这时候就需要引入 polyfill，就是垫片。只要在代码入口的地方引入@babel-polyfill，就会把@babel-polyfill 所有提供的垫片引进来。这样的问题是会使得代码体积大很多，所以不推荐在库中使用 polyfill。  
  我们看最新版本的 polyfill 其实代码如下：

  ```js
  import 'core-js/stable';
  import 'regenerator-runtime/runtime';
  ```

  可以看到其实@babel/polyfill 就是由 core-js 和 regenerator-runtime 组成的，core-js 提供了所有 API 的垫片，regenerator-runtime 提供了对 async/await 语法转化的 helper 支持。  
  @babel/polyfill 在 7.4.4 版本后已经被官方 deprecated，官方推荐使用 core-js@3 + @babel/preset-env，其中配置 preset-env 的 corejs 版本为 3。

- @babel/preset-env  
   相信这个是很多主流项目在安装 babel 后第一个要使用的库。我们知道在 babel 的配置文件中可以使用很多插件来转译特定语法，如@babel/plugin-destructuring, @babel/plugin-arrow-functions，如果每次都安装这些插件很繁琐，所以 babel 还有一种叫 preset 的东西，把一套插件组合起来，就叫成 preset，如@babel/preset-react，@babel/preset-es2015。  
   上面说到的 preset,plugin 都是在确定了在项目中需要支持哪些特定的语法后使用的。但这样会存在一个问题，我们用上面的配置生成出来的编译后代码可能在我们的目标浏览器中可能很多是不必要的，如我们的目标浏览器是 chrome，那我们就可以不用转译包括箭头函数在内的绝大部分 es6 的特性。用 babel 这样编译后使得代码体积大了不少。存在很大部分的浪费。那我们有没有可能根据目标浏览器自动选择需要哪些 plugin 呢？  
   答案就是 preset-env。如名字所示，它可以根据你设置的目标浏览器环境自动选择哪些 plugin。如在.babelrc 或者 webpack 中配置:

  ```js
  {
    "presets": [
        ["env", {
            "targets": {
                "browsers": ["last 2 versions", "safari >= 7"]
            }
        }]
    ]
  }
  ```

  另外需要提及的是 preset 关于垫片引入的 useBuiltIns 三种配置：false, entry, usage。false 则只做语法转化，不会处理任何 API 的垫片问题，entry 是在入口处引入所有的垫片，usage 是根据代码中使用到的垫片按需引入。由于 Babel > 7.4.0 后，官方不推荐使用@babel/polyfill，如果需要垫片则要自己安装 core-js 然后设置 useBuiltIns 为 entry 或者 usage，同时指定 corejs 版本（默认为 2）：

  ```js
   {
    "presets": [
        ["env", {
            "targets": {
                "browsers": ["last 2 versions", "safari >= 7"]
            },
            "useBuiltIns": "usage",
            "corejs": 3
        }]
    ]
  }
  ```

  上面的配置解决了垫片的问题，但还缺少了对于 async/await 语法的兼容。只用上面的配置，没有引入别的 polyfill, 若代码中使用了 async/await 语法，会报错，因为我们没有引入 regenerator-runtime。上面说到，@babel/polyfill 是由 core-js 和 regenerator-runtime 组成的，其实 regenerator-runtime 就是提供了 async/await 语法的 helper 方法。若缺少会导致 babel 编译出来的代码运行有问题。当然我不推荐在代码中直接引入 regenerator-runtime，我们有更优雅的方案。请接着看完。

- @babel/plugin-transform-runtime, @babel/runtime  
   我们知道 babel 在编译过程中主要会处理这几样事情：

  - 语法规范的转换，如 es6 中的 class, 箭头函数等
  - 垫片处理，如 Object.assign, Array.includes 等
  - async/await 语法转换

  babel 在转化过程中会创建很多的 helper 方法，如果我们在不同的文件中使用这种需要转化的代码，就会创建很多重复的 helper 方法，这无疑会使我们的代码体积大很多。那我们是不是可以把这些共有的 helper 方法抽离出来呢？transform-runtime 就是做这个事情。另外我们还需要安装@babel/runtime。

  @babel/runtime 里面包含了：

  - core-js 引用（垫片的转化）
  - 通用的 helper 方法（class，箭头函数等语法）
  - regenerator-runtime 引用（async/await 语法的 helper 方法）

  transform-runtime 会把在转译过程中需要用到 helper 方法的地方改为引入@babel/runtime 里面的方法，这样就可以避免创建重复的 helper 方法。贴一段代码帮助理解吧：

  ```js
  // 源码
  class Point {
  	constructor(x, y) {
  		this.x = x;
  		this.y = y;
  	}
  	getX() {
  		return this.x;
  	}
  }
  let cp = new ColorPoint(25, 8);
  // -----------------------------
  // 编译后
  ('use strict');
  require('core-js/modules/es.object.define-property');
  function _classCallCheck(instance, Constructor) {
  	/* 省略... */
  }
  function _defineProperties(target, props) {
  	/* 省略... */
  }
  function _createClass(Constructor, protoProps, staticProps) {
  	/* 省略... */
  }
  var Point = /*#__PURE__*/ (function() {
  	function Point(x, y) {
  		_classCallCheck(this, Point);
  		this.x = x;
  		this.y = y;
  	}
  	_createClass(Point, [
  		{
  			key: 'getX',
  			value: function getX() {
  				return this.x;
  			}
  		}
  	]);
  	return Point;
  })();
  var cp = new ColorPoint(25, 8);
  ```

  使用 transform-runtime 后：

  ```js
  'use strict';
  var _interopRequireDefault = require('@babel/runtime/helpers/interopRequireDefault');
  var _classCallCheck2 = _interopRequireDefault(
  	require('@babel/runtime/helpers/classCallCheck')
  );
  var _createClass2 = _interopRequireDefault(
  	require('@babel/runtime/helpers/createClass')
  );
  var Point = /*#__PURE__*/ (function() {
  	function Point(x, y) {
  		(0, _classCallCheck2.default)(this, Point);
  		this.x = x;
  		this.y = y;
  	}
  	(0, _createClass2.default)(Point, [
  		{
  			key: 'getX',
  			value: function getX() {
  				return this.x;
  			}
  		}
  	]);
  	return Point;
  })();
  var cp = new ColorPoint(25, 8);
  ```

  另外使用 transform-runtime 替换 polyfill 的好处是避免全局污染。polyfill 会直接修改原型，如果项目用到第三方库可能会有问题。使用 transform-runtime 则是使用 helper 方法，不会污染全局。

### 如何在编写库时正确使用 babel 配置

- 不使用 polyfill。  
  为了减少打包体积，把整个 polyfill 包（大概 87k）引入进来肯定是不现实的。你会说如果我是给自己的项目用，我在自己项目中引入 polyfill，库本身不引用不就好了。这当然是可以，但不方便你开发测试这个库。因为你需要依赖宿主环境。  
  建议使用@babel/runtime + @babel/plugin-transform-runtime。
- 使用@babel/plugin-transform-runtime  
  上面介绍 babel 全家桶也说过了，使用 transform-runtime 可以减小打包体积，因为除了可以抽离库本身用到的 helper 方法/core-js 引用/regenerator-runtime 等（要注意宿主项目需要安装@babel/runtime），还可以和宿主项目共享@babel/runtime 里面的方法。但不绝对，如果你只有一个文件，且用到的新特性不多，那 helper 方法也不会太多，没有必要使用 transform-runtime。
- 在特定环境下把库使用的第三方库 external 出去
  比如你要开发一个 vue 的组件库，你的库中引用了 vue，但打包时候可以把 vue external 出去。因为你能确定在宿主项目中肯定包含了 vue 的依赖。不需要重复打包在库里面了。
- 构建出来 target 优先使用 esmodule  
  webpack4 以后支持了编译时的 tree shaking。而且 webpack 的 tree shaking 只支持 es6 的 module，所以打包出来的代码如果是 esmodule，webpack 构建会只 bundle 目标库中你引用到的暴露出来的方法或者变量。
