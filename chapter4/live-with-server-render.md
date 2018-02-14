## 前言
近年来前端领域发展迅猛，**前后端分离**早已成为业界共识，各类管控系统(to B)上个SPA什么的也不值一提，但唯独**偏展示类的项目**，为了SEO，始终还是需要依赖于服务器端渲染html。

我[过往](https://segmentfault.com/a/1190000004127975)也曾尝试为SPA弥补SEO，但现在看来，效果虽然达到了，但工作量也大大地增加（因为后端用的PHP，不能做到前后同构）。

虽然无法改变*依赖服务器端渲染*这一现实，但我们可以去勇敢地拥抱它，用前端的坚船利炮(aka webpack)，把服务器端模板层给啃下来！

## 前导知识
- webpack的基本使用
- 利用webpack[生成HTML文件](webpack-build-html-page.md)，及，[构建模板布局系统](webpack-layout-system.md)
- 你所在项目后端模板引擎（如我目前使用的Slim框架，模板引擎仅为原生PHP）


## 两个阶段
整个前端项目，以本文主题的视角来看，可以分为两个阶段：

### 纯静态页面开发阶段
在这个阶段里，一切开发都跟静态网站无二致，按UI稿切好页面搞好交互，要用到ajax请求API的也尽管写，跟后端的协作点仅在于API文档。

传统前端的工作也就到这里为止了，但对我们来说，目前的成果并不是我们最终的交付；因此，注意了，在这个阶段我们是可以**“偷懒”**的，比如说，一些明显应该由服务器端循环生成的部分（商品列表、文章列表等），我们写一遍就OK了。

### 动态页面改造阶段
这就是所谓的**“套页面”**，传统来说是由后端来做的，实际上后端也是苦不堪言，毕竟模板不是自己写的，有时还是需要改造一番，而这正是我们前端要大力争取的活。

在这个阶段里，我们的主要工作是按照后端模板引擎的规则来撰写模板变量占位符，当然这里面也不会少了循环输出和逻辑判断，另外也可能需要用到后端定义的一些函数，视项目需求而定。

### 在两个阶段里来回往返
这两个阶段不一定是完全独立的，有需要的话也是可以做到来回往返的。

那什么时候才叫做*“有需要”*呢？举个例子，当你把原先的静态页面都改造成需要后端渲染的页面模板后，却发现后端此时并未准备好相应的模板变量，而你此时又需要对页面的UI部分进行修改，那么你就很被动了，因为改好的这些页面模板根本跑不起来了。有两种解决方案：

- 参考`API mock`的思路，来个`模板变量 mock`，这就相当于一直留在**动态页面改造**阶段了。
- 回到**纯静态页面开发**阶段，让页面不需要后端渲染也能跑起来。具体怎么做呢？
  1. 区分开两个阶段，使用不同的webpack配置。
  2. 在我们构建生成页面的**前端模板**（注意分清与后端模板的区别），判断（判断依据看[这里](../chapter2/webpack-dev-production-environment.md)）本次执行webpack打包是在哪个“阶段”，继而选择是生成静态（且完整）的element，还是带有模板变量占位符的element。这样一来，我们就可以随时选择在不同的阶段（或称环境）里进行开发了。


## 改造开始
本文着重介绍如何将静态页面改造成后端渲染需要的模板。

### 配合后端模板命名规则生成相应模板文件
不同项目因应本身所使用的后端框架或是其它需求，对模板放置的目录结构也会有所不一样，那么，如何构建后端所需要的目录结构呢？

在静态网页阶段，我习惯把html/css/js都按照所属页面归到各自的目录中（公用的css/js也当然是放到公用目录中），看HtmlWebpackPlugin配置：
```javascript
pageArr.forEach((page) => {
  const htmlPlugin = new HtmlWebpackPlugin({
    filename: `${page}/index.html`, // page变量形如'product/index'、'product/detail'
    template: path.resolve(dirVars.pagesDir, `./${page}/html.js`),
    chunks: [page, 'commons/commons'],
    hash: true,
    xhtml: true,
  });
  pluginsConfig.push(htmlPlugin);
});
```

而在改造阶段，则放到后端指定位置：

```javascript
pageArr.forEach((page) => {
  const htmlPlugin = new HtmlWebpackPlugin({
    filename: `../../view/frontend/${page}.php`, // 通过控制相对路径来确定模板的根目录
    template: path.resolve(dirVars.pagesDir, `./${page}/html.js`),
    chunks: [page, 'commons/commons'],
    hash: true,
    xhtml: true,
  });
  pluginsConfig.push(htmlPlugin);
});
```

此时我模板目录结构是这样的：

```bash
│  
├─alert
│      index.php
│      
├─article
│      detail.php
│      index.php
│      
├─index
│      index.php
│      
├─product
│      detail.php
│      index.php
│      
└─user
        edit-password.php
        modify-info.php
```

这里需要注意的是，我的前端项目目录实际上是作为后端目录里的一个子目录来存放的，这样才能依靠相对路径来确定模板文件存放的根目录位置。

### 处理站内链接
对于站内链接，我建议在前端模板里使用一个函数来适配两个阶段：

```javascript
{
  /* 拼接系统内部的URL */
  constructInsideUrl(url, request, urlTail) {
    urlTail = urlTail || '';
    let finalUrl = config.PAGE_ROOT_PATH + url;
    if (!config.IS_PRODUCTION_MODE) {
      finalUrl += '/index.html' + urlTail;
      return finalUrl;
    }
    return `<?php echo cf::constructInsideUrl(array('module' => '${url}'), $isStaticize)?>`;
  },
};
```

在前端模板里这么用：

```ejs
<a href="<%= constructInsideUrl('index/index') %>">
  <img src="<%= require('./logo.png') %>">
</a>
```

这样做，就能分别在静态页面阶段和后端渲染阶段生成相应的超链接。再者，在后端渲染阶段，我们生成出来的也不一定是一个完整的url，可以像我上述代码一样，生成调用后端函数的模板代码，从而灵活满足后端的一些需求（比如说，我的项目有静态化的需求，那么，静态化后的站内链接跟动态渲染的又会有所不同了）。

### 处理模板变量
这一块其实我要说的不多，无非就是按照后端模板引擎的规则，输出变量、循环输出变量、判断条件输出变量、调用后端（模板引擎）函数调整输出变量。

关键是，我们需要拿到一份**模板变量文档**，跟API文档类似，它实际上也是一份前后端的数据协议。有了这份文档，我们才能在后端未完工的情况下，进入**动态页面改造阶段**，并根据其中内容实现`模板变量 mock`。

### 争讨模板布局渲染权
关于利用模板布局系统对多个页面共有的部分实现复用，在[之前的文章](webpack-layout-system.md)里已经提及了，我设计该系统的思路恰恰是来自于后端模板渲染。那么，在前后端均可以实现**模板布局系统**的前提下，我们应如何抉择呢？我的答案是，前端一定要吃下来！

从前端的角度来看：
- 我们在**纯静态页面开发阶段**的产物就已经是一个个完整的页面了，再要拆开并不现实。
- 由于在webpack的辅助下这套**模板布局系统**功能相当强大，因此并没有给整个项目添加额外的成本。

从后端的角度来看：
- 服务器拼接多个HTML代码段本身也是有成本（比如磁盘IO成本）的，倒不如渲染一个完整的页面。
- 在公共组件的分治管理上不会有很大变化，只不过以前是一个一个组件渲染好后再拼在一起，而现在是把各个组件的数据整合在一起来统一渲染罢了。

## 总结
在后端渲染的项目里使用webpack多页应用架构是绝对可行的，可不要给老顽固们吓唬得又回到传统前端架构了。