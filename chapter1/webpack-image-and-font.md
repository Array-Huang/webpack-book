# 听说webpack连图片和字体也能打包？

## 前言
上一篇[《听说webpack连less/css也能打包？》](webpack-css-and-less.md)说到使用loader来加载CSS，这一篇来讲讲如何**笼统**地加载其它类型的资源。为什么强调是**“笼统”**呢？这是因为本文介绍的方法并不针对任何类型的资源，这意味着，什么类型的资源都能用，但效果也都只是有限的。

## 用的什么loader呢？
本文介绍俩loader：file-loader和url-loader。

### `file-loader`
file-loader的主要功能是：把源文件迁移到指定的目录（可以简单理解为从源文件目录迁移到`build`目录），并返回新文件的路径（简单拼接而成）。

file-loader需要传入`name`参数，该参数接受以下变量（以下讨论的前提是：源文件`src/public-resource/imgs/login-bg.jpg`;在根目录内执行`webpack`命令，也就是当前的上下文环境与`src`目录同级）：

- [ext]：文件的后缀名，示例为'jpg'。
- [name]：文件名本身，示例为'login-bg'。
- [path]：相对于当前执行webpack命令的目录的相对路径（不含文件名本身），示例为'src/public-resource/imgs/'。这个参数我感觉用处不大，除非你想把迁移后的文件放回源文件的目录或其子目录里。
- [hash]：源文件内容的hash，用于**缓存解决方案**。

我的做法是，`require('!file-loader?name=static/images/[name].[ext]!../imgs/login-bg.jpg')`，这样`login-bg.jpg`的路径就变成`static/images/login-bg.jpg`了，注意这还不是完整的路径，最终还是要拼上webpack配置中的`output.publicPath`参数的；比如说我的`output.publicPath`参数是`../../../../build/`，那么最终从`require()`里获得的完整路径就会是`../../../../build/static/images/login-bg.jpg`了。

### `url-loader`
url-loader的主要功能是：将源文件转换成DataUrl(声明文件mimetype的base64编码)。据我所知，在前端范畴里，图片和字体文件的DataUrl都是可以被浏览器所识别的，因此可以把图片和字体都转化成DataUrl收纳在HTML/CSS/JS文件里，以减少HTTP连接数。

url-loader主要接受以下参数：

- limit参数，数据类型为整型，表示*目标文件的体积*大于多少**字节**就换用file-loader来处理了，不填则永远不会交给file-loader处理。例如`require("url?limit=10000!./file.png");`，表示如果目标文件大于10000字节，就交给file-loader处理了。
- mimetype参数，前面说了，DataUrl是需要声明文件的mimetype的，因此我们可以通过这个参数来强行设置mimetype，不填写的话则默认从目标文件的后缀名进行判断。例如`require("url?mimetype=image/png!./file.jpg");`，强行把jpg当png使哈。
- 一切file-loader的参数，这些参数会在启用file-loader时传参给file-loader，比如最重要的name参数。

## 实操演示
接下来还是用我的脚手架项目[webpack-seed][3]来介绍如何利用url-loader和file-loader来加载各类资源。

### 图片
这一块我是直接在webpack配置文件里设置的：

```javascript
        {
          // 图片加载器，雷同file-loader，更适合图片，可以将较小的图片转成base64，减少http请求
          // 如下配置，将小于8192byte的图片转成base64码
          test: /\.(png|jpg|gif)$/,
          loader: 'url-loader?limit=8192&name=./static/img/[hash].[ext]',
        },
```

由于使用了[hash]，因此即便是不同页面引用了相同名字但实际内容不同的图片，也不会造成“覆盖”的情况出现；进一步讲，如果不同页面引用了在不同位置但实际内容相同的图片，这还可以归并成一张图片，方便浏览器缓存呢。

### 字体文件
这一块我也还是直接在webpack配置里配置的：

```javascript
        {
          // 专供iconfont方案使用的，后面会带一串时间戳，需要特别匹配到
          test: /\.(woff|woff2|svg|eot|ttf)\??.*$/,
          loader: 'file?name=./static/fonts/[name].[ext]',
        },
```

需要声明的是，由于我使用的是阿里妈妈的iconfont方案，此方案加载字体文件的方式有一点点特殊，所以正则匹配的时候要注意一点，iconfont的CSS是这样的，你们看看就明白了：

```css
@font-face {font-family: "iconfont";
  src: url('iconfont.eot?t=1473142795'); /* IE9*/
  src: url('iconfont.eot?t=1473142795#iefix') format('embedded-opentype'), /* IE6-IE8 */
  url('iconfont.woff?t=1473142795') format('woff'), /* chrome, firefox */
  url('iconfont.ttf?t=1473142795') format('truetype'), /* chrome, firefox, opera, Safari, Android, iOS 4.2+*/
  url('iconfont.svg?t=1473142795#iconfont') format('svg'); /* iOS 4.1- */
}
```

### 其它资源
也许你会问，我们为什么还需要转移其它资源呢？直接引用不就可以了吗？

我之前也是这么做的，直接引用源文件目录`src`里的资源，比如说`webuploader`用到的swf文件，比如说用来兼容IE而又不需要打包的js文件。但是后来我发现，这样做的话，就导致部署上线的时候要把`build`目录和`src`目录同时放上去了；而且由于`build`目录和`src`目录同级，我就只能用`build`目录和`src`目录的上一级目录作为网站的根目录了（因为如果把`build`目录设为网站，用户就读取不到`src`目录了），反正就是各种的不方便。

那么，我是怎么做的呢？

我建了一个config文件，名为`build-file.config.js`，内容如下：

```javascript
module.exports = {
  js: {
    xdomain: require('!file-loader?name=static/js/[name].[ext]!../../../vendor/ie-fix/xdomain.all.js'),
    html5shiv: require('!file-loader?name=static/js/[name].[ext]!../../../vendor/ie-fix/html5shiv.min.js'),
    respond: require('!file-loader?name=static/js/[name].[ext]!../../../vendor/ie-fix/respond.min.js'),
  },
  images: {
    'login-bg': require('!file-loader?name=static/images/[name].[ext]!../imgs/login-bg.jpg'),
  },
};
```

这个config文件起到两个作用：

1. 每次加载到这个config文件的时候，会执行那些`require()`语句，对目标文件进行转移（从`src`目录到`build`目录）。
2. 调用目标文件的代码段，可以从这个config文件取出目标文件转移后的完整路径，例如我在`src/public-resource/components/header/html.ejs`里是这么用的：

```ejs
<!DOCTYPE html>
<html lang="zh-cmn-Hans">
<head>
  <meta http-equiv="X-UA-Compatible" content="IE=edge" />
  <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
  <title><% if (pageTitle) { %> <%= pageTitle %> - <% } %> XXXX后台</title>
  <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1" /> 
  <meta name="renderer" content="webkit" />

  <!--[if lt IE 10]>
    <script src="<%= BUILD_FILE.js.xdomain %>" slave="<%= SERVER_API_URL %>cors-proxy.html"></script>
    <script src="<%= BUILD_FILE.js.html5shiv %>"></script>
  <![endif]-->
</head>
<body>
  <!--[if lt IE 9]>
    <script src="<%= BUILD_FILE.js.respond %>"></script>
  <![endif]-->
```

恩，你可能会好奇这HTML里怎么能直接引用js的值，哈哈哈，超纲了超纲了，这是我后面要讲到的内容了。