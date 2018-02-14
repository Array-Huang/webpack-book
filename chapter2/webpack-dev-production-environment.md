# 开发环境、生产环境傻傻分不清楚？

## 前言
开发环境与生产环境分离的原因如下：

- 在开发时，不可避免会产生大量debug又或是测试的代码，这些代码不应出现在生产环境中（也即不应提供给用户）。
- 在把页面部署到服务器时，为了追求极致的技术指标，我们会对代码进行各种各样的优化，比如说混淆、压缩，这些手段往往会彻底破坏代码本身的可读性，不利于我们进行debug等工作。
- 数据源的差异化，比如说在本地开发时，读取的往往是本地mock出来的数据，而正式上线后读取的自然是API提供的数据了。

如果硬是要在开发环境和生产环境用完全一样的代码，那么必然会付出沉重的代价，这点想必也不用多说了。

下面主要针对两点来介绍如何分离开发环境和生产环境：一是如何以不同的方式进行编译，也即如何分别形成开发环境及生产环境的webpack配置文件；二是在业务代码中如何根据环境的不同而做出不同的处理。

## 如何分离开发环境和生产环境的webpack配置文件
如果同时把一份完整的开发环境配置文件和一份完整的生产环境配置文件列在一起进行比较，那么会出现以下三种情况：

- 开发环境有的配置，生产环境不一定有，比如说开发时需要生成sourcemap来帮助debug，又或是热更新时使用到的`HotModuleReplacementPlugin`。
- 生产环境有的配置，开发环境不一定有，比如说用来混淆压缩js用的`UglifyJsPlugin`。
- 开发环境和生产环境都拥有的配置，但在细节上有所不同，比如说`output.publicPath`，又比如说`css-loader`中的`minimize`和`autoprefixer`参数。

更重要的是，实际上开发环境和生产环境的配置文件的绝大部分都是一致的，对于这一致的部分来说，我们坚决要消除冗余，否则后续维护起来不仅麻烦，而且还容易出错。

### 怎么做呢？
答案很简单：分拆webpack配置文件成N个小module。原先我们是一个完整的配置文件，有好几百行，从头看到尾都头大了，更别说分离不分离的了。下面来看看我分离的结果：

```bash
├─webpack.dev.config.js # 开发环境的webpack配置文件（无实质内容，仅为组织整理）
├─webpack.config.js # 生产环境的webpack配置文件（无实质内容，仅为组织整理）
├─webpack-config # 存放分拆后的webpack配置文件
  ├─entry.config.js # webpack配置中的各个大项，这一级目录里的文件都是
  ├─module.config.js
  ├─output.config.js
  ├─plugins.dev.config.js # 俩环境配置中不一致的部分，此文件由开发环境配置文件webpack.dev.config.js来加载
  ├─plugins.product.config.js # 俩环境配置中不一致的部分，此文件由生产环境配置文件webpack.config.js来加载
  ├─resolve.config.js
  │  
  ├─base # 主要是存放一些变量
  │   ├─dir-vars.config.js
  │   ├─page-entries.config.js
  │      
  ├─inherit # 存放生产环境和开发环境相同的部分，以供继承
  │   ├─plugins.config.js
  │      
  └─vendor # 存放webpack兼容第三方库所需的配置文件
      ├─eslint.config.js
      ├─postcss.config.js
```

文件目录结构看过了，接下来看一下我是如何组织整理最后的配置文件的：

```javascript
/* 开发环境webpack配置文件webpack.dev.config.js */
module.exports = {
  entry: require('./webpack-config/entry.config.js'),

  output: require('./webpack-config/output.config.js'),

  module: require('./webpack-config/module.config.js'),

  resolve: require('./webpack-config/resolve.config.js'),

  plugins: require('./webpack-config/plugins.dev.config.js'),

  eslint: require('./webpack-config/vendor/eslint.config.js'),

  postcss: require('./webpack-config/vendor/postcss.config.js'),
};
```

这样，你就可以很轻松地处理开发/生产环境配置文件中相同与不同的部分了。

### 如何分别调用开发/生产环境的配置文件呢？
还记得我在《[webpack多页应用架构系列（二）：webpack配置常用部分有哪些？][3]》里讲过，我们在控制台调用webpack命令来启动打包时，可以添加上`--config`参数来指定webpack配置文件的路径吗？我们可以配合上`npm scripts`来使用，在package.json里定义：

```json
  "scripts": {
    "build": "node build-script.js && webpack --progress --colors",
    "dev": "node build-script.js && webpack --progress --colors --config ./webpack.dev.config.js",
    "watch": "webpack --progress --colors --watch --config ./webpack.dev.config.js"
  },
```

这样一来，当我们开发的时候就可以使用`npm run dev`或`npm run watch`，而到要上线打包的时候就运行`npm run build`。

## 业务代码如何判断生产/开发环境
在业务代码里要判断生产/开发环境其实很简单，只需一个变量即可：

```javascript
if (IS_PRODUCTION) {
    // 做生产环境该做的事情
} else {
    // 做开发环境该做的事情
}
```

这么一来，关键就在于这变量`IS_PRODUCTION`是怎么来的了。

在我还没分离开发和生产环境时，我用的办法是，开发时在业务代码所使用的配置文件中把这变量设为`false`，而在最后打包上线时就手动改为`true`。这种方法我用过一段时间，非常繁琐，而且经常上线后发现，我嘞个去怎么ajax读的是我本地的mock服务器。

### 怎么做呢？
我参考了许多文章，先粗略讲讲我没有采用的方法：

- 用`EnvironmentPlugin`引入process.env，这样就可以在业务代码中靠`process.env.NODE_ENV`来判断了。
- 用`ProvidePlugin`来控制在不同环境里加载不同的配置文件（业务代码用的）。

那我用的是什么方法呢？我最后选用的是[`DefinePlugin`][4]。

举个官方例子，其大概用法是这样的：

```javascript
new webpack.DefinePlugin({
    PRODUCTION: JSON.stringify(true),
    VERSION: JSON.stringify("5fa3b9"),
    BROWSER_SUPPORTS_HTML5: true,
    TWO: "1+1",
    "typeof window": JSON.stringify("object")
})
```

`DefinePlugin`可能会被误认为其作用是在webpack配置文件中为编译后的代码上下文环境设置全局变量，但其实不然。它真正的机制是：`DefinePlugin`的参数是一个object，那么其中会有一些`key-value`对。在webpack编译的时候，会把业务代码中没有定义（使用var/const/let来预定义的）而变量名又与`key`相同的变量（直接读代码的话的确像是全局变量）替换成`value`。例如上面的官方例子，`PRODUCTION`就会被替换为`true`；`VERSION`就会被替换为`'5fa3b9'`（注意单引号）；`BROWSER_SUPPORTS_HTML5`也是会被替换为`true`；`TWO`会被替换为`1+1`（相当于是一个数学表达式）；`typeof window`就被替换为`'object'`了。

再举个例子，比如你在代码里是这么写的：

```javascript
if (!PRODUCTION)
    console.log('Debug info')
if (PRODUCTION)
    console.log('Production log')
```

那么在编译生成的代码里就会是这样了：

```javascript
if (!true)
    console.log('Debug info')
if (true)
    console.log('Production log')
```

而如果你用了`UglifyJsPlugin`，则会变成这样：

```javascript
console.log('Production log')
```

如此一来，只要在俩环境的配置文件里用`DefinePlugin`分别定义好`IS_PRODUCTION`的值，我们就可以在业务代码里进行判断了：

```javascript
  /* global IS_PRODUCTION:true */
  if (!IS_PRODUCTION) {
    console.log('如果你看到这个Log，那么这个版本实际上是开发用的版本');
  }
```

需要注意的是，如果你在webpack里整合了ESLint，那么，由于ESLint会检测没有定义的变量（ESLint要求使用全局变量时要用`window.xxxxx`的写法），因此需要一个`global`注释声明（`/* global IS_PRODUCTION:true */`）IS_PRODUCTION是一个全局变量(当然在本例中并不是)来规避warning。