## 前言
一个成熟的项目，自然离不开迭代更新；那么在部署前端这一块，我们免不了总是要顾及到浏览器缓存的，本文将介绍如何在 webpack (架构)的帮助下，妥善处理好浏览器缓存。

实际上，我很早以前就想写这一part了，只是苦于当时我所掌握的方案不如人意，便不敢献丑了；而自从
 webpack 升级到 v2 版本后，以及第三方plugin的日益丰富，我们也有了更多的手段来处理cache。

## 浏览器缓存简单介绍
下面来简单介绍一下浏览器缓存，以及为何我要在标题中强调“该去则去，该留则留”。

### 浏览器缓存是啥？
浏览器缓存(Browser Cache)，是浏览器为了节省网络带宽、加快网站访问速度而推出的一项功能。浏览器缓存的运行机制是这样的：

1. 用户使用浏览器第一次访问某网站页面，该页面上引入了各种各样的静态资源（js/css/图片/字体……），浏览器会把这些静态资源，甚至是页面本身(html文件)，都一一储存到本地。
2. 用户在后续的访问中，如果需要再次请求同样的静态资源（根据 url 进行匹配），且静态资源没有过期（服务器端有一系列判别资源是否过期的策略，比如`Cache-Control`、`Pragma`、`ETag`、`Expires`、`Last-Modified`），则直接使用前面本地储存的资源，而不需要重复请求。

由于webpack只负责构建生成网站前端的静态资源，不涉及服务器，因此本文不讨论以*HTTP Header*为基础的缓存控制策略；那我们讨论什么呢？

很简单，由于浏览器是根据静态资源的**url**来判断该静态资源是否已有缓存，而静态资源的文件目录又是相对固定的，那么重点明显就在于静态资源的**文件名**了；我们就通过操控静态资源的文件名，来决定静态资源的“去留”。

### 浏览器缓存，该留不留会怎么样？
每次部署上线新版本，静态资源的文件名若有变化，则浏览器判断是第一次读取这个静态资源；那么，即便这个静态资源的内容跟上一版的完全一致，浏览器也要重新下载这个静态资源，浪费网络带宽、拖慢页面加载速度。 

### 浏览器缓存，该去不去会怎么样？
每次部署上线新版本，静态资源的文件名若没有变化，则浏览器判断可加载之前缓存下来的静态资源；那么，即便这个静态资源的内容跟上一版的有所变化，浏览器也察觉不到，使用了老版本的静态资源。那这会造成什么样的影响呢？可大可小，小至用户看到的依然是老版的资源，达不到上线更新版本的目的；大至造成网站运行报错、布局错位等问题。

## 如何通过操控静态资源的文件名达到控制浏览器缓存的目的呢？
在webpack关于文件名命名的配置中，存在一系列的变量（或者理解成命名规则也可），通过这些变量，我们可以根据所要生成的文件的具体情况来进行命名，而不必预设好一个固定的名称。在缓存处理这一块，我们主要用到`[hash]`和`[chunkhash]`这两个变量。关于这两个变量的介绍，我在之前的文章 —— [《webpack配置常用部分有哪些？》](../chapter1/webpack-config-introduction.md)就已经解释过是什么意思了，这里就不再累述。

这里总结下`[hash]`和`[chunkhash]`这两个变量的用法：
- 用`[hash]`的话，由于每次使用 webpack 构建代码的时候，此 hash 字符串都会更新，因此相当于**强制刷新浏览器缓存**。
- 用`[chunkhash]`的话，则会根据具体 chunk 的内容来形成一个 hash 字符串来插入到文件名上；换句说， chunk 的内容不变，该 chunk 所对应生成出来的文件的文件名也不会变，由此，**浏览器缓存便能得以继续利用**。

## 有哪些资源是需要兼顾浏览器缓存的？
理论上来说，除了HTML文件外（HTML文件的路径需要保持相对固定，只能从服务器端入手），webpack生成的所有文件都需要处理好浏览器缓存的问题。

### js
在 webpack 架构下，js文件也有不同类型，因此也需要不同的配置：

1. 入口文件(Entry)：在webpack配置中的`output.filename`参数中，让生成的文件名中带上`[chunkhash]`即可。
2. 异步加载的chunk：`output.chunkFilename`参数，操作同上。
3. 通过`CommonsChunkPlugin`生成的文件：在`CommonsChunkPlugin`的配置参数中有`filename`这一项，操作同上。但需要注意的是，如果你使用`[chunkhash]`的话，webpack 构建的时候可是会报错的哦；那可咋办呢，用`[hash]`的话，这`common chunk`不就每次上线新版本都强制刷新了吗？这其实是因为，webpack 的 runtime && manifest 会统一保存在你的`common chunk`里，解决的方法，就请看下面关于“webpack 的 runtime && manifest”的部分了。

### css
对于css来说，如果你是用`style-loader`直接把css内联到`<head>`里的，那么，你管好引入该css的js文件的浏览器缓存就好了。

而如果你是使用`extract-text-webpack-plugin`把css独立打包成css文件的，那么在文件名的配置上，~~同样加上`[chunkhash]`即可~~加上`[contenthash]`即可(感谢@FLYiNg_hbt 提醒)。这个`[contenthash]`是什么东西呢？其实就是`extract-text-webpack-plugin`为了与`[chunkhash]`区分开，而自定义的一个命名规则，其实际含义跟`[chunkhash]`可以说是一致的，只是`[chunkhash]`已被占用作为 chunk 的内容 hash 字符串了，继续用`[chunkhash]`会造成“文件改动监测失败”的问题。

### 图片、字体文件等静态资源
如[《听说webpack连图片和字体也能打包？》](../chapter1/webpack-image-and-font.md)里介绍的，处理这类静态资源一般使用`url-loader`或`file-loader`。

对于`url-loader`来说，就不需要关心浏览器缓存了，因为它是把静态资源转化成 dataurl 了，而并非独立的文件。

而对于`file-loader`来说，同样是在文件名的配置上加上`[chunkhash]`即可。另外需要注意的是，`url-loader`一般搭配有降级到`file-loader`的配置（使用loader加载的文件大于一个你设定的值就降级到使用`file-loader`来加载），同样需要在文件名的配置上加上`[chunkhash]`。

### webpack 的`runtime && manifest`
所谓的runtime，就是帮助 webpack 编译构建后的打包文件在浏览器运行的一些辅助代码段，换句话说，打包后的文件，除了你自己的源码和npm库外，还有 webpack 提供的一点辅助代码段。

而 manifest，则是 webpack 用以查找 chunk 真实路径所使用的一份关系表，简单来说，就是** chunk 名**对应** chunk 路径**的关系表。manifest 一般来说会被藏到 runtime 里，因此我们查看 runtime 的时候，虽然能找得到 manifest，但一般都不那么直观，形如下面这一段（仅`common chunk`部分）：

```javascript
u.type = "text/javascript", u.charset = "utf-8", u.async = !0, u.timeout = 12e4, n.nc && u.setAttribute("nonce", n.nc), u.src = n.p + "" + e + "." + {
    0: "e6d1dff43f64d01297d3",
    1: "7ad996b8cbd7556a3e56",
    2: "c55991cf244b3d833c32",
    3: "ecbcdaa771c68c97ac38",
    4: "6565e12e7bad74df24c3",
    5: "9f2774b4601839780fc6"
}[e] + ".bundle.js";
```

#### `runtime && manifest`被打包到哪里去了？
那么，这`runtime && manifest`的代码段，会被放到哪里呢？一般来说，如果没有使用`CommonsChunkPlugin`生成`common chunk`，`runtime && manifest`会被放在以入口文件为首的chunk（俗称“大包”）里，如果是我们这种多页（又称多入口）应用，则会每个大包一份`runtime && manifest`；这夸张的冗余我们自然是不能忍的，那么
用上`CommonsChunkPlugin`后，`runtime && manifest`就会统一迁到`common chunk`了。

#### `runtime && manifest`给`common chunk`带来的缓存危机
虽说把`runtime && manifest`迁到`common chunk`后，代码冗余的问题算是解决了，但却造成另一问题：由于我们在上述的静态资源的文件名命名上都采用了`[chunkhash]`的方案，因此也使得只要我们稍一改动源代码，就会有起码一个 chunk 的命名会产生变化，这就会导致我们的`runtime && manifest`也产生变化，从而导致我们的`common chunk`也发生变化，这或许就是 webpack 规定含有`runtime && manifest`的`common chunk`不能使用`[chunkhash]`的原因吧（反正chunkhash肯定会变的，还不如不用呢是不是）。

要解决上述问题（这问题很严重啊我摔，`common chunk`怎么能用不上缓存啊，这可是最大的chunk啊），我们就需要把`runtime && manifest`给独立出去。方法也很简单，在用来打包`common chunk`的`CommonsChunkPlugin`后，再加一`CommonsChunkPlugin`：

```javascript
  /* 抽取出所有通用的部分 */
  new webpack.optimize.CommonsChunkPlugin({
    name: 'commons/commons',      // 需要注意的是，chunk的name不能相同！！！
    filename: '[name]/bundle.[chunkhash].js', // 由于runtime独立出去了，这里便可以使用[chunkhash]了
    minChunks: 4,
  }),
  /* 抽取出webpack的runtime代码，避免稍微修改一下入口文件就会改动commonChunk，导致原本有效的浏览器缓存失效 */
  new webpack.optimize.CommonsChunkPlugin({
    name: 'webpack-runtime',
    filename: 'commons/commons/webpack-runtime.[hash].js', // 注意runtime只能用[hash]
  }),
```

这样一来，`runtime && manifest`代码段就会被打包到这个名为`webpack-runtime`的 chunk 里了。这是什么原理呢？据说是在使用`CommonsChunkPlugin`的情况下， webpack 会把`runtime && manifest`打包到最后面的一个`CommonsChunkPlugin`生成的 chunk 里，而如果这个chunk没有其它代码，那么自然就达到了把`runtime && manifest`独立出去的目的了。

需要注意的是，如果你用了`html-webpack-plugin`来生成html页面，记得要把这`runtime && manifest`的 chunk 插入到html页面上，不然页面报错了可不怪我哦。

至此，由于`runtime && manifest`独立出去成一个chunk了，于是`common chunk`的命名便可以使用`[chunkhash]`了，也就是说，`common chunk`现在也能做到公共模块内容有更新了，才更新文件名；另一方面，这个独立出去的 `runtime && manifest` chunk，是每次 webpack 打包构建的时候都会更新了。

#### 有必要把 manifest 从 `runtime && manifest` chunk 中独立出去吗？
是的，不用惊讶，的确是有这么一个骚操作。

把 manifest 独立出去的理由是这样的：manifest 独立出去后，runtime 的部分基本上就不会有变动了；到这里，我们就知道，`runtime && manifest`里实际上就是 manifest 在变；因此把 manifest 独立出去，也是进一步地利用浏览器缓存（可以把 runtime 的缓存保留下来）。

具体是怎么做的呢？主流有俩方案：

- 利用[chunk-manifest-webpack-plugin](https://github.com/soundcloud/chunk-manifest-webpack-plugin)把 manifest 生成一个json文件，然后由 webpack 异步加载。
- 如果你是用`html-webpack-plugin`来生成html页面的话，还可以利用[inline-chunk-manifest-html-webpack-plugin](https://github.com/jouni-kantola/inline-chunk-manifest-html-webpack-plugin)（`html-webpack-plugin`作者推荐）来把manifest直接输出到html页面上，这样就能省一个 Http 请求了。

我试用过第二种方案，好使，但最终还是放弃了，为什么呢？

把 manifest 独立出去后，只剩下 runtime 的 chunk 的命名还是只能用`[hash]`，而不能利用`[chunkhash]`，这就导致我们根本没法利用浏览器缓存。后来，我又想出一个折衷的办法，连`[hash]`也不要了，直接写死一个文件名；这样的话，的确浏览器缓存就能保存下来了。但后来我还是反转了自己，这种方法虽然能留下浏览器缓存，却做不到“该去则去”。或许大家会有疑问，你不是说 runtime 不会变的吗，那留下缓存有什么关系呀？是的，在同一 webpack 环境下 runtime 的确不会变，但难保 webpack 环境改变后，这runtime会怎么样呀。比如说 webpack 的版本升级了、 webpack 的配置改了、loader & plugin 的版本升级了，在这些情况下，谁敢保证 runtime 永远不会变啊？这 runtime 一用错了过期的缓存，那很可能整个系统都会崩溃的啊，这个险我实在是冒不起，所以只能作罢。

不过我看了下[Array-Huang/webpack-seed](https://github.com/Array-Huang/webpack-seed)的`runtime && manifest` chunk，也才 2kb 而已嘛，你们管好自己的强迫症和代码洁癖好吗？！

## 缓存问题杂项

### 模块id带来的缓存问题
webpack 处理模块(module)间依赖关系时，需要给各个模块定一个 id 以作标识。webpack 默认的 id 命名规则是根据模块引入的顺序，赋予一个整数(1、2、3……)。当你在源码中任意增添或删减一个模块的依赖，都会对整个
 id 序列造成极大的影响，可谓是“牵一发而动全身”了。那么这对我们的浏览器缓存会有什么样直接的影响呢？影响就是会造成，各个chunk中都不一定有实质的变化，但引用的依赖模块id却都变了，这明显就会造成 chunk 的文件名的变动，从而影响浏览器缓存。

webpack 官方文档里推荐我们使用一个已内置进 webpack2 里的 plugin：`HashedModuleIdsPlugin`，这个 plugin 的官方文档在[这里](https://webpack.js.org/plugins/hashed-module-ids-plugin/#components/sidebar/sidebar.jsx)。

webpack1 时代便有一个`NamedModulesPlugin`，它的原理是直接使用模块的相对路径作为模块的 id，这样只要模块的相对路径，模块 id 也就不会变了。那么这个`HashedModuleIdsPlugin`对比起`NamedModulesPlugin`来说又有什么进步呢？

是这样的，由于模块的相对路径有可能会很长，那么就会占用大量的空间，这一点是一直为社区所诟病的；但这个`HashedModuleIdsPlugin`是根据模块的相对路径生成(默认使用md5算法)一个长度可配置（默认截取4位）的字符串作为模块的 id，那么它占用的空间就很小了，大家也就可以安心服用了。

> To generate identifiers that are preserved over builds, webpack supplies the NamedModulesPlugin (recommended for development) and HashedModuleIdsPlugin (recommended for production).

从上可知，官方是推荐开发环境用`NamedModulesPlugin`，而生产环境用`HashedModuleIdsPlugin`的，原因似乎是与热更新(hmr)有关；不过就我看来，仅在生产环境用`HashedModuleIdsPlugin`就行了，开发环境还管啥浏览器缓存啊，俺开 chrome dev-tool 设置了不用任何浏览器缓存的。

用法也挺简单的，直接加到`plugin`参数就成了：

```javascript
plugins: {
  // 其它plugin
  new webpack.HashedModuleIdsPlugin(),  
}
```

### 由某些 plugin 造成的文件改动监测失败
有些 plugin 会生成独立的 chunk 文件，比如`CommonsChunkPlugin`或`ExtractTextPlugin`（从js中提取出css代码段并生成独立的css文件） 。

这些 plugin 在生成 chunk 的文件名时，可能没料想到后续还会有其它 plugin （比如用来混淆代码的`UglifyJsPlugin`）会对代码进行修改，因此，由此生成的 chunk 文件名，并不能完全反映文件内容的变化。

另外，`ExtractTextPlugin`有个比较严重的问题，那就是它生成文件名所用的`[chunkhash]`是直接取自于引用该css代码段的 js chunk ；换句话说，如果我只是修改 css 代码段，而不动 js 代码，那么最后生成出来的css文件名依然没有变化，这可算是非常严重的浏览器缓存“该去不去”问题了。
2017-07-26 改动：改用`[contenthash]`便不会出现此问题，上见***css部分***。

有一款 plugin 能解决以上问题：[webpack-plugin-hash-output](https://github.com/scinos/webpack-plugin-hash-output)。

> There are other webpack plugins for hashing out there. But when they run, they don't "see" the final form of the code, because they run before plugins like webpack.optimize.UglifyJsPlugin. In other words, if you change webpack.optimize.UglifyJsPlugin config, your hashes won't change, creating potential conflicts with cached resources.

> The main difference is that webpack-plugin-hash-output runs in the last compilation step. So any change in webpack or any other plugin that actually changes the output, will be "seen" by this plugin, and therefore that change will be reflected in the hash.

简单来说，就是这个`webpack-plugin-hash-output`会在 webpack 编译的最后阶段，重新对所有的文件取文件内容的 md5 值，这就保证了文件内容的变化一定会反映在文件名上了。

用法也比较简单：

```javascript
plugins: {
  // 其它plugin
  new HashOutput({
    manifestFiles: 'webpack-runtime', // 指定包含 manifest 在内的 chunk
  }),
}
```

## 总结
浏览器缓存很重要，很重要，很重要，出问题了怕不是要给领导追着打。另外，这一块的细节特别多，必须方方面面都顾到，不然哪一方面出了纰漏就全局泡汤。