# 怎么打包公共代码才能避免重复？

## 前言
与单页应用相比，多页应用存在多个入口（每个页面即一个入口），每一个入口（页面）都意味着一套完整的js代码（包括业务逻辑和加载的第三方库/框架等）。
在文章[《webpack配置常用部分有哪些？》](../chapter1/webpack-config-introduction.md)中，我介绍了如何配置多页应用的入口(entry)，然而，如果仅仅如此操作，带来的后果就是，打包生成出来的每一个入口文件都会完整包含所有代码。
你也许会说："咱们以前写页面不也是每个页面都会加载所有的代码吗？浏览器会缓存，没事的啦"。其实问题在于，以前写代码都是单个单个js来加载的，一个页面加载下来的确所有页面都能共享到缓存；而到了webpack这场景，由于对于每一个页面来说，所有的js代码都打包成唯一一个js文件了，而浏览器是无法分辨出该文件内的公共代码并加以缓存的，所以，浏览器就没办法实现公共代码在页面间的缓存了（当前页面的缓存还是OK的，也就是说刷新不需要重新加载）。

## 想智能判断并打包公共代码？CommonsChunkPlugin能帮到你
CommonsChunkPlugin的效果是：在你的多个页面（入口）所引用的代码中，找出其中满足条件（被多少个页面引用过）的代码段，判定为公共代码并打包成一个独立的js文件。至此，你只需要在每个页面都加载这个公共代码的js文件，就可以既保持代码的完整性，又不会重复下载公共代码了（多个页面间会共享此文件的缓存）。

### 再提一下使用Plugin的方法
大部分Plugin的使用方法都有一个固定的套路：

1. 利用Plugin的初始方法并传入Plugin预设的参数进行初始化，生成一个实例。
2. 将此实例插入到webpack配置文件中的`plugins`参数（数组类型）里即可。

### CommonsChunkPlugin的初始化常用参数有哪些？
- `name`，给这个包含公共代码的chunk命个名（唯一标识）。
- `filename`，如何命名打包后生产的js文件，也是可以用上`[name]`、`[hash]`、`[chunkhash]`这些变量的啦（具体是什么意思，请看我上一篇文章中关于filename的那一节）。
- `minChunks`，公共代码的判断标准：某个js模块被多少个chunk加载了才算是公共代码。
- `chunks`，表示需要在哪些chunk（也可以理解为webpack配置中entry的每一项）里寻找公共代码进行打包。不设置此参数则默认提取范围为所有的chunk。

### 实例分析
实例来自于我的脚手架项目`webpack-seed`，我是这样初始化一个CommonsChunkPlugin的实例：

```javascript
  var commonsChunkPlugin = new webpack.optimize.CommonsChunkPlugin({
    name: 'commons', // 这公共代码的chunk名为'commons'
    filename: '[name].bundle.js', // 生成后的文件名，虽说用了[name]，但实际上就是'commons.bundle.js'了
    minChunks: 4, // 设定要有4个chunk（即4个页面）加载的js模块才会被纳入公共代码。这数目自己考虑吧，我认为3-5比较合适。
  });
```

最终生成文件的路径是根据webpack配置中的ouput.path和上面CommonsChunkPlugin的filename参数来拼的，因此想控制目录结构的，直接在filename参数里动手脚即可，例如：`filename: 'commons/[name].bundle.js'`

## 总结
整体来说，这套方案还是相当简单的，而从效果上说，也算是比较均衡的，比较适合项目初期使用。