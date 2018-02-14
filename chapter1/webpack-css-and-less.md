# 听说webpack连less/css也能打包？

## 前言
过去讲前端模块化、组件化，更多还是停留在js层面，毕竟js作为一种更典型的程序语言，在这方面的想象和操作空间都更大一些。但近年来，组件化要求得更多了，HTML/CSS/JS这三件套一件可都不能少（甚至包括其它类型的资源，比如说图片），而这样的组件，无疑是`高内聚`的。

## 文章简介
本文将介绍如何使用webpack来打包less/css（没用过sass，但毕竟也是通过loader来加载的，相信与less无异），首先是介绍相关的webpack plugin&loader，然后将介绍如何加载不同应用层次的less/css。

## 用到什么loader了？
在[《webpack多页应用架构系列（二）：webpack配置常用部分有哪些？》][2]里我就说过，webpack的核心只能打包js文件，而js以外的资源都是靠loader进行转换或做出相应的处理的。下面我就来介绍打包less/css所需要的loader。

### less-loader
针对less文件，我们首先需要使用less-loader来加载。less-loader会调用所依赖的`less`模块对less文件进行编译(包括`@import`语法)。至于说less-loader所接受的参数，实质上大部分是传递给`less`模块使用的参数，由于我本人应用less的程度不深，因此没有传任何参数、直接就使用了。如果你之前对`less`模块就已经有了一套配置的话，请参考[less-loader的文档][3]进行配置。

另外，less-loader并不会针对`url()`语法做特别的转换，因此，如果你想把`url()`语句里涉及到的文件（比如图片、字体文件等）也一并用webpack打包的话，就必须利用管道交给css-loader做进一步的处理。

### css-loader
针对css文件，我们需要使用css-loader来加载.css-loader的功能比较强大，一些新颖的特性比如`Local Scope`或是`CSS Modules`都是支持的。

我目前只用到了css-loader的[压缩功能(Minification)][4]，对于这个功能，有一点是需要注意的，那就是如果你的代码里也和我一样，有许多为了浏览器兼容性的**废弃**CSS代码的话，请务必关闭`autoprefixer`已避免你的**废弃**CSS代码被css-loader删除了，形如`css?minimize&-autoprefixer`。

上面提到css-loader会对`url()`语句做处理，这里稍微再说两句。在less/css里的这`url()`语句，在css-loader看来，就跟`require()`语句是一样的，只要在webpack配置文件里定义好加载各类型资源的loader，那这`url()`语句实际上什么资源都能处理。一般我在`url()`语句都会以相对路径的方式（相对于此语句所在的less/css文件）来指定文件路径；请不要使用以`/`开头（即相对于网站根目录，因为对于文件系统来说，这明显是令人混淆的）的路径，尽管css-loader也可以通过设置`root`参数来适配。

### postcss-loader
习惯用postcss的童鞋们有福啦，webpack可以通过postcss-loader来兼容postcss。由于postcss只算是一个加分项，因此这里也不作过多介绍，只介绍一下如何把postcss撘进webpack，不明白的童鞋麻烦先把postcss搞懂了再看。

放上我的脚手架项目的代码：

```javascript
var precss       = require('precss');
var autoprefixer = require('autoprefixer');

module.exports = {
    module: {
        loaders: [
            {
                test:   /\.css$/,
                exclude: /node_modules|bootstrap/,
                loader: 'css?minimize&-autoprefixer!postcss',
            }
        ]
    },
    postcss: function () {
        return [precss, autoprefixer({
            remove: false,
            browsers: ['ie >= 8', '> 1% in CN'],
        })];
    }
}
```

从loader的配置`'css?minimize&-autoprefixer!postcss'`上看，实际上就是先让postcss-loader处理完了再传递给css-loader。而`postcss`项则是postcss-loader所接受的参数，实际上就是返回一个包含你所需要的postcss's plugins的数组啦，这些plugin有各自的初始化参数，不过这些都是postcss的内容了，这里就不做介绍了。

## 用到什么Plugin了？
加载less/css这一块主要用到的是`extract-text-webpack-plugin`（下文简称为`ExtractTextPlugin`吧），而且由于我用的是`webpack 1`，因此用的也是相对应`webpack 1`的版本（[1的文档在这里不要搞错了哈][5]）。

ExtractTextPlugin的作用是把各个chunk加载的css代码（可能是由less-loader转换过来的）合并成一个css文件并在页面加载的时候以`<link>`的形式进行加载。

相对于使用style-loader直接把css代码段跟js打包在一起并在页面加载时以inline的形式插入DOM，我还是更喜欢ExtractTextPlugin生成并加载CSS文件的形式；倒不是看不惯inline的css，只是用文件形式来加载的话会快很多，尤其后面介绍用webpack来生成HTML的时候，这`<link>`会直接生成在`<head>`里，那么在CSS的加载上就跟传统的前端页面没有差别了，体验非常棒。

ExtractTextPlugin的初始化参数不多，唯一的必填项是`filename`参数，也就是如何来命名生成的CSS文件。跟webpack配置里的output.filename参数类似，这ExtractTextPlugin的filename参数也允许使用变量，包括[id]、[name]和[contenthash]；理论上来说如果只有一个chunk，那么不用这些变量，写死一个文件名也是可以的，但由于我们要做的是多页应用，必然存在多个chunk（至少每个entry都对应一个chunk啦）。这里我是这么设置的：

```javascript
new ExtractTextPlugin('[name]/styles.css'), [name]对应的是chunk的name，我在webpack配置中是这样
```

`[name]`对应的是chunk的name，我在webpack配置中把各个entry的name都按`index/index`、`index/login`这样的形式来设置了，那么最后css的路径就会像这样：`build/index/index/styles.css`，也就是跟chunk的js文件放一块了（js文件的路径形如`build/index/index/entry.js`）。

除了要把这初始化后的ExtractTextPlugin放到webpack配置中的`plugins`参数里，我们还要在loader配置里做相应的修改：

```javascript
module.exports = {
    module: {
        loaders: [
            {
                test:   /\.css$/,
                exclude: /node_modules|bootstrap/,
                loader: ExtractTextPlugin.extract('css?minimize&-autoprefixer!postcss'),
            }
        ]
    },
}
```

如此一来，ExtractTextPlugin就算是配置好了。

##如何加载不同应用层次的less/css
在我的设计中，有三种应用层次的less/css代码段：

- 基础的、公用的代码段，包括CSS框架、在CSS框架上进行定制的CSS theme，基本上每个页面都会应用到这些CSS代码段。
- 组件的代码段，这里的`组件`指的是你自己写的组件，而且组件本身含有js，并负责加载css以及其它逻辑。
- 每个页面独有的CSS代码段，很可能只是对某些细节进行微调。

首先来回顾一下我设计的文件目录结构：

```bash
├─src # 当前项目的源码
    ├─pages # 各个页面独有的部分，如入口文件、只有该页面使用到的css、模板文件等
    │  ├─alert # 业务模块
    │  │  └─index # 具体页面
    │  ├─index # 业务模块
    │  │  ├─index # 具体页面
    │  │  └─login # 具体页面
    │  │      └─templates # 如果一个页面的HTML比较复杂，可以分成多块再拼在一起
    │  └─user # 业务模块
    │      ├─edit-password # 具体页面
    │      └─modify-info # 具体页面
    └─public-resource # 各个页面使用到的公共资源
        ├─components # 组件，可以是纯HTML，也可以包含js/css/image等，看自己需要
        │  ├─footer # 页尾
        │  ├─header # 页头
        │  ├─side-menu # 侧边栏
        │  └─top-nav # 顶部菜单
        ├─config # 各种配置文件
        ├─iconfont # iconfont的字体文件
        ├─imgs # 公用的图片资源
        ├─layout # UI布局，组织各个组件拼起来，因应需要可以有不同的布局套路
        │  ├─layout # 具体的布局套路
        │  └─layout-without-nav # 具体的布局套路
        ├─less # less文件，用sass的也可以，又或者是纯css
        │  ├─base-dir
        │  ├─components-dir # 如果组件本身不需要js的，那么要加载组件的css比较困难，我建议可以直接用less来加载
        │  └─base.less # 组织所有的less文件
        ├─libs # 与业务逻辑无关的库都可以放到这里
        └─logic # 业务逻辑
```

### 基础代码段
基础的CSS代码（实际上我的项目中用的都是less）我统一都放到`src/public-resource/less`目录里。我使用一个抽象的文件`base.less`将所有的less文件组织起来（利用`@import`），这样的话我用js加载起来就方便多了。

在我的脚手架项目（[Array-Huang/webpack-seed][6]）里，CSS框架我用的是bootstrap，并且使用了bootstrap-loader进行加载，因此就没有把bootstrap的CSS文件放到`src/public-resource/less/base-dir`目录里，这个目录里放的都是我定制的theme了。

`src/public-resource/less/components-dir`目录放的是某些第三方组件所用到的css，又或是不含js的组件所用到的css。其实这部分CSS是否应该归在下一类，我也考虑良久，只是由于归到下一类的话加载起来不方便，不方便原因如下：

- 某些第三方库是要你自己加载CSS的，如果你打算写适配器来封装这些第三方库，那自然可以直接在适配器来加载CSS，这就属于下一类了；然而，有一些使用起来很简单的库，你还写适配器那就有点画蛇添足了。
- 某些自己写的组件可能仅包含HTML和CSS，那么谁来加载CSS？

所以干脆还是交由`base.less`一并加载了算了。

我设计了一个`common.page.js`，并在每一个页面的入口文件里都首先加载这`common.page.js`，那么，只要我在这`common.page.js`里加载`base.less`，所有的页面都能享受到这份基础CSS代码段。

### 组件代码段
组件的代码我都放在了`src/public-resource/components`，每一个组件统一放在一个独立的目录，并由该组件的js负责加载其CSS。

### 页面代码段
页面独有的CSS我自然是放在该页面自己的目录里，利用该页面的入口文件进行加载。

### 最终生成的CSS代码都在哪？
由于我使用了ExtractTextPlugin，因此这些CSS代码最终都会生成到所属chunk的目录里成为一个CSS文件。

- 基础代码段肯定是保存在CommonsChunkPlugin所生成的公共代码chunk所在的目录里了，在我的脚手架项目（[Array-Huang/webpack-seed](https://github.com/Array-Huang/webpack-seed)）里就是`build/commons`了（我的公共代码chunk的name是'commons'）。
- 组件代码段看情况，该组件用的页面多的话（大于CommonsChunkPlugin的minChunks参数）就会被归到跟基础代码段一起咯；反之，则哪个页面用到它，就放到哪个页面chunk的目录里咯。
- 页面代码段就不用想了，肯定是在那个页面chunk的目录里了，毕竟才用了1次。