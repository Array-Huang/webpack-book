# 利用webpack生成HTML普通网页&页面模板

## 为什么要用webpack来生成HTML页面
按照我们前面的十一篇的内容来看，自己写一个HTML页面，然后在上面加载webpack打包的js或其它类型的资源，感觉不也用得好好的么？

是的没错，不用webpack用requireJs其实也可以啊，甚至于，传统那种人工管理模块依赖的做法也没有什么问题嘛。

但既然你都已经看到这一篇了，想必早已和我一样，追求着以下这几点吧：

- 更懒，能自动化的事情绝不做第二遍。
- 更放心，调通的代码比人靠谱，更不容易出错。
- 代码洁癖，什么东西该放哪，一点都不能含糊，混在一起我就要死了。

那么，废话不多说，下面就来说说使用webpack生成HTML页面有哪些好处吧。

### 对多个页面共有的部分实现复用
在实际项目的开发过程中，我们会发现，虽然一个项目里会有很多个页面，但这些页面总有那么几个部分是相同或相似的，尤其是页头页尾，基本上是完全一致的。那我们要怎么处理这些共有的部分呢？

#### 复制粘贴流

> 不就是复制粘贴的事嘛？写好一份完整的HTML页面，做下个页面的时候，直接copy一份文件，然后直接在copy的文件上进行修改不就好了吗？

谁是这么想这么做的，放学留下来，我保证不打死你！我曾经接受过这么一套系统，顶部栏菜单想加点东西，就要每个页面都改一遍，可维护性烂到爆啊。

#### Iframe流
Iframe流常见于管理后台类项目，可维护性OK，就是缺陷比较多，比如说：

- 点击某个菜单，页面是加载出来了但是浏览器地址栏上的URL没变，刷新的话又回到首页了。
- 搜索引擎收录完蛋，前台项目一般不能用Iframe来布局。
- 没有逼格，Low爆了，这是最重要的一点（大误）。

#### SPA流
最近这几年，随着移动互联网的兴起，SPA也变得非常常见了。不过SPA的局限性也非常大，比如搜索引擎无法收录，但我个人最在意的，是它太复杂了，尤其是一些本来业务逻辑就多的系统，很容易懵圈。

#### 后端模板渲染
这倒真是一个办法，只是，需要后端的配合，利用后端代码把页面的各个部分给拼合在一起，所以这方法对前端起家的程序员还是有点门槛的。

#### **利用前端模板引擎生成HTML页面**
所谓“用webpack生成HTML页面”，其实也并不是webpack起的核心作用，实际上靠的还是前端的模板引擎将页面的各个部分给拼合在一起来达到公共区域的复用。webpack更多的是组织统筹整个生成HTML页面的过程，并提供更大的控制力。最终，webpack生成的到底是完整的页面，还是供后端渲染的模板，就全看你自己把控了，非常灵活，外人甚至察觉不出来这到底是你自己写的还是代码统一生成的。

### 处理资源的动态路径
如果你想用**在文件名上加hash**的方法作为缓存方案的话，那么用webpack生成HTML页面就成为你唯一的选择了，因为随着文件的变动，它的hash也会变化，那么整个文件名都会改变，你总不能在每次编译后都手动修改加载路径吧？还是放心交给webpack吧。

### 自动加载webpack生成的css、less
如果你使用webpack来生成HTML页面，那么，你可以配置好每个页面加载的chunk（webpack打包后生成的js文件），生成出来的页面会自动用`<script>`来加载这些chunk，路径什么的你都不用管了哈（当然前提是你配置好了output.publicPath）。另外，用`extract-text-webpack-plugin`打包好的css文件，webpack也会帮你自动添加到`<link>`里，相当方便。

### 彻底分离源文件目录和生成文件目录
使用webpack生成出来的HTML页面可以很安心地跟webpack打包好的其它资源放到一起，相对于另起一个目录专门存放HTML页面文件来说，整个文件目录结构更加合理：

```bash
build
  - index
    - index
      - entry.js
      - page.html
    - login
      - entry.js
      - page.html
      - styles.css
```

## 如何利用webpack生成HTML页面
webpack生成HTML页面主要是通过[`html-webpack-plugin`](https://github.com/ampedandwired/html-webpack-plugin)来实现的，下面来介绍如何实现。

### `html-webpack-plugin`的配置项
每一个html-webpack-plugin的对象实例都只针对/生成一个页面，因此，我们做多页应用的话，就要配置多个html-webpack-plugin的对象实例：

```javascript
pageArr.forEach((page) => {
  const htmlPlugin = new HtmlWebpackPlugin({
    filename: `${page}/page.html`,
    template: path.resolve(dirVars.pagesDir, `./${page}/html.js`),
    chunks: [page, 'commons'],
    hash: true, // 为静态资源生成hash值
    minify: true,
    xhtml: true,
  });
  configPlugins.push(htmlPlugin);
});
```

`pageArr`实际上是各个chunk的name，由于我在output.filename设置的是`'[name]/entry.js'`，因此也起到构建文件目录结构的效果（具体请看[这里](https://segmentfault.com/a/1190000006863968#articleHeader5)），附上`pageArr`的定义：

```javascript
module.exports = [
  'index/login',
  'index/index',
  'alert/index',
  'user/edit-password', 'user/modify-info',
];
```

`html-webpack-plugin`的配置项真不少，这里仅列出多页应用常用到的配置：

- filename，生成的网页HTML文件的文件名，注意可以利用`/`来控制文件目录结构的，其最终生成的路径，是基于webpack配置中的output.path的。
- template，指定一个基于某种模板引擎语法的模板文件，`html-webpack-plugin`默认支持**ejs**格式的模板文件，如果你想使用其它格式的模板文件，那么需要在webpack配置里设置好相应的loader，比如`handlebars-loader`啊`html-loader`啊之类的。如果不指定这个参数，`html-webpack-plugin`会使用一份默认的ejs模板进行渲染。如果你做的是简单的SPA应用，那么这个参数不指定也行，但对于多页应用来说，我们就依赖模板引擎给我们拼装页面了，所以这个参数非常重要。
- inject，指示把加载js文件用的`<script>`插入到哪里，默认是插到`<body>`的末端，如果设置为'head'，则把`<script>`插入到`<head>`里。
- minify，生成压缩后的HTML代码。
- hash，在**由`html-webpack-plugin`负责加载的js/css文件**的网址末尾加个URL参数，此URL参数的值是代表本次编译的一个hash值，每次编译后该hash值都会变化，属于缓存解决方案。
- chunks，以数组的形式指定由`html-webpack-plugin`负责加载的chunk文件（打包后生成的js文件），不指定的话就会加载所有的chunk。

### 生成一个简单的页面
下面提供一份供生成简单页面（之所以说简单，是因为不指定页面模板，仅用默认模板）的配置：

```javascript
var HtmlWebpackPlugin = require('html-webpack-plugin');
var webpackConfig = {
  entry: 'index.js',
  output: {
    path: 'dist',
    filename: 'index_bundle.js'
  },
  plugins: [new HtmlWebpackPlugin({
    title: '简单页面',
    filename: 'index.html',
  })],
};
```

使用这份配置编译后，会在dist目录下生成一个index.html，内容如下所示：

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>简单页面</title>
  </head>
  <body>
    <script src="index_bundle.js"></script>
  </body>
</html>
```

由于没有指定模板文件，因此生成出来的HTML文件仅有最基本的HTML结构，并不带实质内容。可以看出，这更适合React这种把HTML藏js里的方案。

### 利用模板引擎获取更大的控制力
接下来，我们演示如何通过制定模板文件来生成HTML的内容，由于`html-webpack-plugin`原生支持ejs模板，因此这里也以ejs作为演示对象：

```ejs
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta http-equiv="Content-type" content="text/html; charset=utf-8"/>
    <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1" /> 
    <title><%= htmlWebpackPlugin.options.title %></title>
  </head>
  <body>
    <h1>这是一个用<b>html-webpack-plugin</b>生成的HTML页面</h1>
    <p>大家仔细瞧好了</p>
  </body>
</html>
```

'html-webpack-plugin'的配置里也要指定**template**参数：

```javascript
var HtmlWebpackPlugin = require('html-webpack-plugin');
var webpackConfig = {
  entry: 'index.js',
  output: {
    path: 'dist',
    filename: 'index_bundle.js'
  },
  plugins: [new HtmlWebpackPlugin({
    title: '按照ejs模板生成出来的页面',
    filename: 'index.html',
    template: 'index.ejs',
  })],
};
```

那么，最后生成出来的HTML文件会是这样的：

```html
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta http-equiv="Content-type" content="text/html; charset=utf-8"/>
    <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1" /> 
    <title>按照ejs模板生成出来的页面</title>
  </head>
  <body>
    <h1>这是一个用<b>html-webpack-plugin</b>生成的HTML页面</h1>
    <p>大家仔细瞧好了</p>
    <script src="index_bundle.js"></script>
  </body>
</html>
```

到这里，我们已经可以控制整个HTML文件的内容了，那么生成后端渲染所需的模板也就不是什么难事了，以PHP的模板引擎smarty为例：

```ejs
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta http-equiv="Content-type" content="text/html; charset=utf-8"/>
    <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1" /> 
    <title><%= htmlWebpackPlugin.options.title %></title>
  </head>
  <body>
    <h1>这是一个用<b>html-webpack-plugin</b>生成的HTML页面</h1>
    <p>大家仔细瞧好了</p>
    <p>这是用smarty生成的内容：<b>{$articleContent}</b></p>
  </body>
</html>
```

### 处理资源的动态路径
接下来在上面例子的基础上，我们演示如何处理资源的动态路径：

```javascript
var HtmlWebpackPlugin = require('html-webpack-plugin');
var webpackConfig = {
  entry: 'index.js',
  output: {
    path: 'dist',
    filename: 'index_bundle.[chunkhash].js'
  },
  plugins: [new HtmlWebpackPlugin({
    title: '按照ejs模板生成出来的页面',
    filename: 'index.html',
    template: 'index.ejs',
  })],
  module: {
    loaders: {
      // 图片加载器，雷同file-loader，更适合图片，可以将较小的图片转成base64，减少http请求
      // 如下配置，将小于8192byte的图片转成base64码
      test: /\.(png|jpg|gif)$/,
      loader: 'url?limit=8192&name=./static/img/[hash].[ext]',
    },
  },
};
```

```ejs
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta http-equiv="Content-type" content="text/html; charset=utf-8"/>
    <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1" /> 
    <title><%= htmlWebpackPlugin.options.title %></title>
  </head>
  <body>
    <h1>这是一个用<b>html-webpack-plugin</b>生成的HTML页面</h1>
    <p>大家仔细瞧好了</p>
    <img src="<%= require('./imgs/login-bg.jpg')  %>" />
  </body>
</html>
```

我们改动了什么呢？

1. 参数`output.filename`里，我们添了个变量[chunkhash]，这个变量的值会随chunk内容的变化而变化，那么，这个chunk文件最终的路径就会是一个动态路径了。
2. 我们在页面上添加了一个`<img>`，它的src是require一张图片，相应地，我们配置了针对图片的loader配置，如果图片比较小，`require()`就会返回DataUrl，而如果图片比较大，则会拷贝到`dist/static/img/`目录下，并返回新图片的路径。

下面来看看，到底`html-webpack-plugin`能不能处理好这些动态的路径。

```html
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta http-equiv="Content-type" content="text/html; charset=utf-8"/>
    <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1" /> 
    <title>按照ejs模板生成出来的页面</title>
  </head>
  <body>
    <h1>这是一个用<b>html-webpack-plugin</b>生成的HTML页面</h1>
    <p>大家仔细瞧好了</p>
    <img src="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEAYABgAAD/2wBDAAgGBgcGBQgHBwcJCQgKDBQNDAsLDBkSEw8UHRofHh0aHBwgJC4nICIsIxwcKDcpLDAxNDQ0Hyc5PTgyPC4zNDL/2wBDAQkJCQwLDBgNDRgyIRwhMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjL/wAARCAAaAFADASIAAhEBAxEB/8QAHwAAAQUBAQEBAQEAAAAAAAAAAAECAwQFBgcICQoL/8QAtRAAAgEDAwIEAwUFBAQAAAF9AQIDAAQRBRIhMUEGE1FhByJxFDKBkaEII0KxwRVS0fAkM2JyggkKFhcYGRolJicoKSo0NTY3ODk6Q0RFRkdISUpTVFVWV1hZWmNkZWZnaGlqc3R1dnd4eXqDhIWGh4iJipKTlJWWl5iZmqKjpKWmp6ipqrKztLW2t7i5usLDxMXGx8jJytLT1NXW19jZ2uHi4+Tl5ufo6erx8vP09fb3+Pn6/8QAHwEAAwEBAQEBAQEBAQAAAAAAAAECAwQFBgcICQoL/8QAtREAAgECBAQDBAcFBAQAAQJ3AAECAxEEBSExBhJBUQdhcRMiMoEIFEKRobHBCSMzUvAVYnLRChYkNOEl8RcYGRomJygpKjU2Nzg5OkNERUZHSElKU1RVVldYWVpjZGVmZ2hpanN0dXZ3eHl6goOEhYaHiImKkpOUlZaXmJmaoqOkpaanqKmqsrO0tba3uLm6wsPExcbHyMnK0tPU1dbX2Nna4uPk5ebn6Onq8vP09fb3+Pn6/9oADAMBAAIRAxEAPwD29Y4fJkm1BxJLCkYuHZHSHdH8+9EYkDls5BJ4ALEpxxjfETw1c3McU9tdSaW0ssZubq0MkZkbuHZiQu1nBXbnDgfKBg7niC3uJfCmqyQQ3dvLLBKsiTT+YwVTIwKqC6/MTjAwdjAZBRVHiFxNZnwFZxgxPfR6jIQox5iRsiZPqFJAz67e5FcVfEShPTok/W7tY4sViJ0pJQ03/D7j6J+0XEtzHDCsSr5SyyysHZeWGFTgK2QJOd2V+QlSGrB1LxFqh1+40/SvD76ibDY8ki34gAZ1OAQR8wwenPODjIFafhvY2jWUnlwCQ2duA6Nlnj2Aru44+Yvgcjvnkgc9c+HbaefW9ZbxTqFtEJXeYafKEEOxcEPt3FiAoOOCM9ATXVU3tH+rHqYZxtzVFbRd+vprsdHpd9dyqovNKubW7mbfPF53mxwghgp3khSD5YyqZILgkDcWqyLpbHTZrnUJFght/MZ5JDgLGpOGJ3N/CAck5PUgHgYvhLU7m80m8kupWvb2zlktmaNgvniNm2sFJCqWyRnjOOTxxz/jXWLifxLYaXPoGtahoCIt3K2nWhmS6lzlI2OQuwYDEZOTtzgZzrSjGrJW7X+RNWEoSaa2f9f0zY8G+Im8T6PqGpW9ggubW8ure2+0M6F1LB1DFgzRg5QEDIG3gYAUa2u6/p3hTTGurx5GBcssKuGlfc43bQzDIG7OM4A4HYVwvwl1xDDr9odC1GwRL+4uiBat5MQ+UeSuACXX+4FBwBx2q/8AGJHbwvaMqkqt4NxA6fI1XjoqhN8q7foPCU1WrKE31aNnQ/iDo2u6p/ZsaXVrdlQyR3UYUvldwxgn+Eg84yCMZrS/tJdOura1nH2ayjs7mWSW7l3MqwtGodnLH5SrFiWOcYzg5FeTQXlvd/EXwuYLiKbbbWqMUbOGCjI+ozyO1eo6syzeIYrZHTzv7OuIgjXLW5Z5SpjVXX5gSIJTlASoQn0zjGTcbef5G+IoQpzVuqv6GrfwXs0Z+xXv2eTy3X5ow4JI+VuehBx6jBYEHII8lvPBXizVoE0660iwikF2ZH1OIwxq64xyiAMecnJGeegr1Pw3LJP4X0iaaRpJZLKFndzlmJQEknua06xqUI1XeX9df6sebiKCq+7J2tfYpW+mRW8Onp5k5NlGI4yszKrfLt+ZQdrcdNwODyMVhaj4FtL/AFS6vItS1LTlugpmj0+4MQlcE5Z+oORgYwO+c7q6SSztpYriKS3heO5BE6MgIlyoU7h/F8oA57DFTV0SjzJSl5m9NujpDQzdNtLfRLOHS7aAhUSR4lhiYLtDDgsSRu+YdWBY7iBgHFmeGNpoy9u84kZQ2WBSPZl1cqxwPmwMqCclewyLNYXhj/iZ+BNG+3/6X9q0yDz/AD/n83dEN27P3s5Oc9c1UYX1+X5icmSWGgafo0N6Astwl7qBvXWVBJtmZ1IIAXgKwUgn7uMk8ZqDxfp+sX+gzQ6NND9oLhmhuIo3SVAOUw6kdcHnv3Aq5qsskeo6IqSOqyXrK4VsBh9nmOD6jIB+oFalZzbqaSY6dTknzJapnlmg+DtbuvFWm6tqOlwaTDZRR7o43iPmuoOSFj+Vcnk/XjNdlDCqeL727S3YyQaTbokCLHuIMkx2gnofkAxuC+vQEdDRUqCW39XNqmJlUfvLpY//2Q==" />
    <script src="index_bundle.c3a064486c8318e5e11a.js"></script>
  </body>
</html>
```

显然，`html-webpack-plugin`成功地将chunk加载了，又处理好了转化为DataUrl格式的图片，这一切，都是我们手工难以完成的事情。

## 还未结束
至此，我们实现了使用webpack生成HTML页面并尝到了它所带来的甜头，但我们尚未实现**对多个页面共有的部分实现复用**，下一篇[《构建一个简单的模板布局系统》](webpack-layout-system.md)我们就来介绍这部分的内容。