# 总有刁民想害朕！ESLint为你阻击垃圾代码

## 前言

### 刁民，还不退下？啊……来人啊快救驾！
你所在的团队里有没有“老鼠屎”？就是专门写各种看起来溜得飞起但实际上晦涩难懂的代码？又或是缩进换行乱成一团？

你写代码是不是特粗心？经常落下些语法错误，debug起来想死？

如果你有以上问题，ESLint帮到你！[手动滑稽]

### ESLint的用途是？
从上面两个应用场景，你大概已经猜到ESLint是用来干什么的了：

- 审查代码是否符合编码规范和统一的代码风格；
- 审查代码是否存在语法错误；

语法错误好说，编码规范和代码风格如何审查呢？ESLint定义好了一大堆规则作为可配置项;同时，一些大公司会开源出来他们使用的配置（比如说`airbnb`），你可以在某套现成配置的基础上进行修改，修改成适合你们团队使用的编码规范和代码风格。

### 本文主要讲什么？
本文着重介绍如何在webpack里整合进ESLint，而并不介绍ESLint本身，因此，对于没有使用过ESLint的小伙伴，请先去自己入门一下啦。

## webpack如何整合ESLint？
这次我们需要使用到`eslint-loader`，先放出配置的代码：

```javascript
/* 这是webpack配置文件的内容，省略无关部分 */
{
  module: {
    preLoaders: [{
      test: /\.js$/, // 只针对js文件
      loader: 'eslint', // 指定启用eslint-loader
      include: dirVars.srcRootDir, // 指定审查范围仅为自己团队写的业务代码
      exclude: [/bootstrap/], // 剔除掉不需要利用eslint审查的文件
    }],
  },
  eslint: {
    configFile: path.resolve(dirVars.staticRootDir, './.eslintrc'), // 指定eslint的配置文件在哪里
    failOnWarning: true, // eslint报warning了就终止webpack编译
    failOnError: true, // eslint报error了就终止webpack编译
    cache: true, // 开启eslint的cache，cache存在node_modules/.cache目录里
  }
}
```

接下来解释一下这份eslint-loader的配置。

### 为嘛把eslint-loader放在`preLoaders`而不是`loaders`里？
理论上来说放loaders里也无伤大雅，但放preLoaders里有以下好处：

- 放在preLoader是先于loader的，因此当ESLint审查到问题报了warning/error的时候就会停掉，可以稍微省那么一点点时间吧大概[手动滑稽]。
- 如果你使用了babel，或类似的loader，那么，通过webpack编译前后的代码相差就很大了，这会造成两个问题（以babel为例）：
  - babel把你的代码转成什么样你自己是无法控制的，这往往导致无法通过ESLint的审查。
  - 我们实际上并不关心编译后生成的代码，我们只需要管好我们自己手写的代码即可，反正谁也不会没事去读读编译后的代码吧？

### 如何传参给eslint-loader？
从[eslint-loader官方文档](https://github.com/MoOx/eslint-loader)可以看出，eslint-loader的配置还是比较多也比较复杂的，因此采用了独立的一个配置项`eslint`（跟`module`同级哈）。

## 总结
只要你能在自己团队里成功推行ESLint，那么最起码，你可以放心不用再看到那些奇奇怪怪的代码了，因为，它们都编译不通过呐哈哈哈哈哈……

## 后话
通过webpack整合ESLint，我们可以保证编译生成的代码都是没有语法错误且符合编码规范的；但在开发过程中，等到编译的时候才察觉到问题可能也是太慢了点儿。

因此我建议可以把ESLint整合进编辑器或IDE里，像我本人在用`Sublime Text 3`的，就可以使用一个名为`SublimeLinter`的插件，一写了有**问题**的代码，就马上会标识出来，如下图所示：

![SublimeLinter效果图][1]


  [1]: SublimeLinter%E6%95%88%E6%9E%9C%E5%9B%BE.jpg "SublimeLinter效果图.jpg"