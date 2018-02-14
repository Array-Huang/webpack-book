# 预打包Dll，实现webpack音速编译

## 前言
书承上文[《如何打造一个自定义的bootstrap》](webpack-build-bootstrap.md)。

上文说到我们利用webpack来打包一个可配置的bootstrap，但文末留下一个问题：由于bootstrap十分庞大，因此每次编译都要耗费大部分的时间在打包bootstrap这一块，而换来的仅仅是配置的便利，十分不划算。

我也并非是故意卖关子，这的确是我自己开发中碰到的问题，而在撰写完该文后，我立即着手探索解决之道。终于，发现了webpack这一大杀器：`DllPlugin`&`DllReferencePlugin`，打包时间过长的问题得到完美解决。

## 解决方案的机制和原理
`DllPlugin`&`DllReferencePlugin`这一方案，实际上也是属于代码分割的范畴，但与[CommonsChunkPlugin](../chapter2/webpack-common-chunk.md)不一样的是，它不仅仅是把公用代码提取出来放到一个独立的文件供不同的页面来使用，它更重要的一点是：把公用代码和它的使用者（业务代码）从编译这一步就分离出来，换句话说，我们可以分别来编译公用代码和业务代码了。这有什么好处呢？很简单，业务代码常改，而公用代码不常改，那么，我们在日常修改业务代码的过程中，就可以省出编译公用代码那一部分所耗费的时间了（是不是马上就联想到坑爹的bootstrap了呢）。

整个过程大概是这样的：

1. 利用`DllPlugin`把公用代码打包成一个**“Dll文件”**（其实本质上还是js，只是套用概念而已）；除了Dll文件外，`DllPlugin`还会生成一个manifest.json文件作为公用代码的索引供`DllReferencePlugin`使用。
2. 在业务代码的webpack配置文件中配置好`DllReferencePlugin`并进行编译，达到利用`DllReferencePlugin`让业务代码和Dll文件实现关联的目的。
3. 在各个页面<head>中，先加载Dll文件，再加载业务代码文件。


### 适用范围
Dll文件里只适合放置不常改动的代码，比如说第三方库（谁也不会有事无事就升级一下第三方库吧），尤其是本身就庞大或者依赖众多的库。如果你自己整理了一套成熟的框架，开发项目时只需要在上面添砖加瓦的，那么也可以把这套框架也打包进Dll文件里，甚至可以做到多个项目共用这一份Dll文件。

### 如何配置哪些代码需要打包进Dll文件？
我们需要专门为Dll文件建一份webpack配置文件，不能与业务代码共用同一份配置：

```javascript
const webpack = require('webpack');
const ExtractTextPlugin = require('extract-text-webpack-plugin');
const dirVars = require('./webpack-config/base/dir-vars.config.js'); // 与业务代码共用同一份路径的配置表

module.exports = {
  output: {
    path: dirVars.dllDir,
    filename: '[name].js',
    library: '[name]', // 当前Dll的所有内容都会存放在这个参数指定变量名的一个全局变量下，注意与DllPlugin的name参数保持一致
  },
  entry: {
    /*
      指定需要打包的js模块
      或是css/less/图片/字体文件等资源，但注意要在module参数配置好相应的loader
    */
    dll: [
      'jquery', '!!bootstrap-webpack!bootstrapConfig',
      'metisMenu/metisMenu.min', 'metisMenu/metisMenu.min.css',
    ],
  },
  plugins: [
    new webpack.DllPlugin({
      path: 'manifest.json', // 本Dll文件中各模块的索引，供DllReferencePlugin读取使用
      name: '[name]',  // 当前Dll的所有内容都会存放在这个参数指定变量名的一个全局变量下，注意与参数output.library保持一致
      context: dirVars.staticRootDir, // 指定一个路径作为上下文环境，需要与DllReferencePlugin的context参数保持一致，建议统一设置为项目根目录
    }),
    /* 跟业务代码一样，该兼容的还是得兼容 */
    new webpack.ProvidePlugin({
      $: 'jquery',
      jQuery: 'jquery',
      'window.jQuery': 'jquery',
      'window.$': 'jquery',
    }),
    new ExtractTextPlugin('[name].css'), // 打包css/less的时候会用到ExtractTextPlugin
  ],
  module: require('./webpack-config/module.config.js'), // 沿用业务代码的module配置
  resolve: require('./webpack-config/resolve.config.js'), // 沿用业务代码的resolve配置
};
```

### 如何编译Dll文件？
编译Dll文件的代码实际上跟编译业务代码是一样的，记得利用`--config`指定上述专供Dll使用的webpack配置文件就好了：

```bash
$ webpack --progress --colors --config ./webpack-dll.config.js
```

另外，建议可以把该语句写到`npm scripts`里，好记一点哈。

### 如何让业务代码关联Dll文件？
我们需要在供编译业务代码的webpack配置文件里设好`DllReferencePlugin`的配置项：

```javascript
new webpack.DllReferencePlugin({
  context: dirVars.staticRootDir, // 指定一个路径作为上下文环境，需要与DllPlugin的context参数保持一致，建议统一设置为项目根目录
  manifest: require('../../manifest.json'), // 指定manifest.json
  name: 'dll',  // 当前Dll的所有内容都会存放在这个参数指定变量名的一个全局变量下，注意与DllPlugin的name参数保持一致
});
```

配置好`DllReferencePlugin`了以后，正常编译业务代码即可。不过要注意，必须要先编译Dll并生成manifest.json后再编译业务代码；而以后每次修改Dll并重新编译后，也要重新编译一下业务代码。

### 如何在业务代码里使用Dll文件打包的module/资源？
不需要刻意做些什么，该怎么require就怎么require，webpack都会帮你处理好的了。

### 如何整合Dll？
在每个页面里，都要按这个顺序来加载js文件：Dll文件 => `CommonsChunkPlugin`生成的公用chunk文件（如果没用`CommonsChunkPlugin`那就忽略啦） => 页面本身的入口文件。

有两个注意事项：

- 如果你是像我一样利用`HtmlWebpackPlugin`来生成HTML并自动加载chunk的话，请务必在`<head>`里手写`<script>`来加载Dll文件。
- 为了完全分离源文件和编译后生成的文件，也为了方便在编译前可以清空build目录，不应直接把Dll文件编译生成到build目录里，我建议可以先生成到源文件src目录里，再用`file-loader`给原封不动搬运过去。

## 光说不练假把式，来个跑分啊大兄弟！
下面以我的脚手架项目[Array-Huang/webpack-seed](https://github.com/Array-Huang/webpack-seed)为例，测试一下（使用开发环境的webpack配置文件`webpack.dev.config.js`）使用这套Dll方案前后的webpack编译时间：

- 使用Dll方案前的编译时间为：10秒17
- 使用Dll方案后的编译时间为：4秒29

由于该项目只是一个脚手架，涉及到的第三方库并不多，我只把jQuery、bootstrap、metisMenu给打包进Dll文件里了，尽管如此，还是差了将近6秒了，相信在实际项目中，这套`DllPlugin`&`DllReferencePlugin`的方案能为你省下更多的时间来找女朋友（大误）。