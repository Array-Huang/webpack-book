# 教练我要写ES6！webpack怎么整合Babel？

## 前言
> 一直以来，我对ES6都不甚感兴趣，一是因为在生产环境中使用ES5已是处处碰壁，其次则是只当这ES6是语法糖不曾重视。只是最近学习react生态，用起babel来转换jsx之余，也不免碰到诸多用上ES6的教程、案例，因此便稍作学习。这一学习，便觉得这语法糖实在是甜，忍不住尝鲜，于是记录部分自觉对自己有用的方法在此。

这是我数月前的一篇文章[《ES6部分方法点评（一）》](https://segmentfault.com/a/1190000005042668)中的一段，如今再看我自己的代码，触目皆是ES6的语法。在当前的浏览器市场下，想在生产环境用上ES6，Babel是必不可少的。

由于我本身只用了ES6的语法而未使用ES6的其它特性，因此本文只介绍如何利用webpack整合Babel来编译ES6的语法，而实际上若要使用ES6的其它属性甚至是ES7（ES2016），其实只需要引入Babel其它的preset/plugin即可，在用法上并无多大变化。

## 用到哪些npm包？
首先要说到的是`babel-loader`，这是webpack整合Babel的关键，我们需要配置好babel-loader来加载那些使用了ES6语法的js文件；换句话说，那些本来就是ES5语法的文件，其实是不需要用babel-loader来加载的，用了也只会浪费我们编译的时间。

然后就是babel相关的npm包，其中包括：
- `babel-core`，babel的核心，没啥好说的。
- `babel-preset-es2015-loose`，babel的preset（相当于是一整套plugin）。babel是有许多preset的，看自己需要来选用，比如说我只管ES6（ES2016）语法的就可以用`babel-preset-es2015`或`babel-preset-es2015-loose`。这俩preset其实用法一样，差别就在于：
> 许多Babel的插件有两种模式：

> 尽可能符合ECMAScript6语义的normal模式和提供更简单ES5代码的loose模式。

> 优点：生成的代码可能更快，对老的引擎有更好的兼容性，代码通常更简洁，更加的“ES5化”。

> 缺点：你是在冒险——随后从转译的ES6到原生的ES6时你会遇到问题。

我自己的考虑是，肯定要更好的兼容性和更好的性能啦这还用想的吗？（敲黑板）

- `babel-plugin-transform-runtime`和`babel-runtime`，这属于优化项，不用也没啥问题，下文会细说。

## 如何配置babel-loader
babel-loader的配置并不复杂，与其它loader并无二致：

```javascript
    {
      test: /\.js$/,
      exclude: /node_modules|vendor|bootstrap/,
      loader: 'babel-loader?presets[]=es2015-loose&cacheDirectory&plugins[]=transform-runtime',
    },
```

下面来详细解释此配置：

- `test: /\.js$/`表明我只用babel-loader来加载js文件，如果你只是小部分js文件应用了ES6，那么也可以给这些文件换个`.es6`的后缀名并把此处改为`test: /\.es6$/`。
- `exclude: /node_modules|vendor|bootstrap/`，上文已经说到了，不需要用babel来加载的文件还是剔除掉，否则会大量增加编译的时间，一般我们只用babel编译我们自己写的应用代码。
- `loader: 'babel-loader?presets[]=es2015-loose&cacheDirectory&plugins[]=transform-runtime'`，这一行是指定使用babel-loader并传入所需参数，这些参数其实也是可以通过babel配置文件.babelrc，不过我还是推荐在这里以参数的方式传入。下面来介绍这些参数：

### preset参数：`babel-preset-es2015-loose`
上文已经解释过preset是什么以及为啥要使用`babel-preset-es2015-loose`了，这里不再累述。

### cacheDirectory参数
cacheDirectory参数默认为false，若你设置为一个文件目录路径（表示把cache存到哪），或是保留为空（表示操作系统默认的缓存目录），则相当于开启cache。这里的cache指的是babel在编译过程中某些可以缓存的步骤，具体是什么我也不太清楚，反正是只要开启了cache就可以加快webpack整体编译速度。我测试了一下，未开启cache的时候我的[脚手架项目(Array-Huang/webpack-seed)](https://github.com/Array-Huang/webpack-seed)需要15秒半来编译；而开启cache后的第一次编译时间并没有减少，第二次编译则变为14秒了，足足减少了1秒半了棒棒哒。

### plugins参数
虽说一个preset已经包括N个plugin了，但总有一些漏网之鱼是要专门加载的。这里我只用到了`transform-runtime`，这个plugin的效果是：不用这plugin的话，babel会为每一个转换后的文件（在webpack这就是每一个chunk了）都添加一些辅助的方法（仅在需要的情况下）；而如果用了这个plugin，babel会把这些辅助的方法都集中到一个文件里统一加载统一管理，算是一个减少冗余，增强性能的优化项吧，用不用也看自己需要了；如果不用的话，前面也不需要安装`babel-plugin-transform-runtime`和`babel-runtime`了。