# 老式jQuery插件还不能丢，怎么兼容？

## 前言
目前前端虽处于百花齐放阶段，angular/react/vue竞相角逐，但毕竟尚未完全成熟，有些需求还是得依靠我们的老大哥jQuery的。

我个人对jQuery并不反感，但我对jQuery生态的停滞不前相当无奈，比如说赫赫有名的bootstrap（特指3代），在webpack上打包还得靠个loader的，太跟不上时势了。况且，bootstrap还算好的，有些jquery插件都有一两年没更新了，连NPM都没上架呢，可偏偏就是找不到它们的替代品，项目又急着要上，这可咋办呐？

别急，今天就教你适配兼容老式jQuery插件。

## 老式jQuery插件为和不能直接用webpack打包？
如果你把jQuery看做是一个普通的js模块来加载（要用到jQuery的模块统统先require后再使用），那么，当你加载老式jQuery插件时，往往会提示找不到jQuery实例（有时候是提示找不到`$`），这是为啥呢？

要解释这个问题，就必须先稍微解释一下jQuery插件的机制：jQuery插件是通过jQuery提供的`jQuery.fn.extend(object)`和`jQuery.extend(object)`这俩方法，来把插件本身实现的方法挂载到`jQuery`（也即`$`）这个对象上的。传统引用jQuery及其插件的方式是先用`<script>`加载jQuery本身，然后再用同样的方法来加载其插件；jQuery会把`jQuery`对象设置为全局变量（当然也包括了`$`），既然是全局变量，那么插件们很容易就能找到`jQuery`对象并挂载自身的方法了。

而webpack作为一个遵从模块化原则的构建工具，自然是要把各模块的上下文环境给分隔开以减少相互间的影响；而jQuery也早已适配了AMD/CMD等加载方式，换句话说，我们在require `jQuery`的时候，实际上并不会把`jQuery`对象设置为全局变量。说到这里，问题也很明显了，jQuery插件们找不到`jQuery`对象了，因为在它们各自的上下文环境里，既没有局部变量`jQuery`（因为没有适配AMD/CMD，所以就没有相应的require语句了），也没有全局变量`jQuery`。

## 怎么来兼容老式jQuery插件呢？
方法有不少，下面一个一个来看。

### `ProvidePlugin` + `expose-loader`
首先来介绍我最为推荐的方法：`ProvidePlugin` + `expose-loader`，在我公司的项目，以及我个人的脚手架开源项目`webpack-seed`里使用的都是这一种方法。

ProvidePlugin的配置是这样的：

```javascript
  var providePlugin = new webpack.ProvidePlugin({
    $: 'jquery',
    jQuery: 'jquery',
    'window.jQuery': 'jquery',
    'window.$': 'jquery',
  });
```

ProvidePlugin的机制是：当webpack加载到某个js模块里，出现了未定义且名称符合（字符串完全匹配）配置中key的变量时，会自动require配置中value所指定的js模块。

如上述例子，当某个老式插件使用了`jQuery.fn.extend(object)`，那么webpack就会自动引入`jquery`（此处我是用NPM的版本，我也推荐使用NPM的版本）。

另外，使用ProvidePlugin还有个好处，就是，你自己写的代码里，再！也！不！用！require！jQuery！啦！毕竟少写一句是一句嘛哈哈哈。

接下来介绍expose-loader，这个loader的作用是，将指定js模块export的变量声明为全局变量。下面来看下expose-loader的配置：

```javascript
/*
    很明显这是一个loader的配置项，篇幅有限也只能截取相关部分了
    看不明白的麻烦去看本系列的另一篇文章《webpack多页应用架构系列（二）：webpack配置常用部分有哪些？》：https://segmentfault.com/a/1190000006863968
 */
{
  test: require.resolve('jquery'),  // 此loader配置项的目标是NPM中的jquery
  loader: 'expose?$!expose?jQuery', // 先把jQuery对象声明成为全局变量`jQuery`，再通过管道进一步又声明成为全局变量`$`
},
```

你或许会问，有了ProvidePlugin为嘛还需要expose-loader？问得好，如果你所有的jQuery插件都是用webpack来加载的话，的确用ProvidePlugin就足够了；但理想是丰满的，现实却是骨感的，总有那么些需求是只能用`<script>`来加载的。

### externals
externals是webpack配置中的一项，用来将某个全局变量**“伪装”**成某个js模块的exports，如下面这个配置：

```javascript
    externals: {
      'jquery': 'window.jQuery',
    },
```

那么，当某个js模块显式地调用`var $ = require('jquery')`的时候，就会把`window,jQuery`返回给它。

与上述`ProvidePlugin + expose-loader`的方案相反，此方案是先用`<script>`加载的jQuery满足老式jQuery插件的需要，再通过externals将其转换成符合模块化要求的exports。

我个人并不太看好这种做法，毕竟这就意味着jQuery脱离NPM的管理了，不过某些童鞋有其它的考虑，例如为了加快每次打包的时间而把jQuery这些比较大的第三方库给分离出去（直接调用公共CDN的第三方库？），也算是有一定的价值。

### imports-loader
这个方案就相当于手动版的ProvidePlugin，以前我用requireJS的时候也是用的类似的手段，所以我一开始从requireJS迁移到webpack的时候用的也是这种方法，后来知道有ProvidePlugin就马上换了哈。

这里就不详细说明了，放个例子大家看看就懂：

```javascript
// ./webpack.config.js

module.exports = {
    ...
    module: {
        loaders: [
            {
                test: require.resolve("some-module"),
                loader: "imports?$=jquery&jQuery=jquery", // 相当于`var $ = require("jquery");var jQuery = require("jquery");`
            }
        ]
    }
};
```

## 总结
以上的方案其实都属于shimming，并不特别针对jQuery，请举一反三使用。另外，上述方案并不仅用于shimming，比如用上`ProvidePlugin`来写少几个require，自己多多挖掘，很有乐趣的哈~~

## 补充

### 误用externals（2016-10-17更新）
有童鞋私信我，说用了我文章的方案依然提示`$ is not a function`，在我仔细分析后，发现：
1. 他用的是我推荐的**`ProvidePlugin` + `expose-loader`**方案，也就是说，他已经把jquery打包进来了。
2. 但是他又不明就里得配了externals：

```javascript
  externals: {
    jquery: 'window.jQuery',
  },
```

3. 然而实际上他并没有直接用`<script>`来引用jQuery，因此window.jQuery是个null。
4. 结果，他的jquery插件获得的`$`就是个null了。

这里面我们可以看出，externals是会覆盖掉`ProvidePlugin`的。

但这里有个问题，`expose-loader`的作用就是设置好window.jQuery和window.$，那window.jQuery怎么会是null呢？我的猜想是：externals在`expose-loader`设置好`window.jQuery`前就已经取了`window.jQuery`的值(`null`)了。

说了这么多，其实关键意思就是，不要手贱不要手贱不要手贱（重要的事情说三遍）！