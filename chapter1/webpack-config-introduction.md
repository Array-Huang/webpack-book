# webpack配置常用部分有哪些？

## 前言
webpack的配置文件是一个node.js的module，用CommonJS风格来书写，形如：
```javascript
module.exports = {
    entry: "./entry",
    output: {
        path: __dirname + "/dist",
        filename: "bundle.js"
    }
}
```
webpack的配置文件并没有固定的命名，也没有固定的路径要求，如果你直接用`webpack`来执行编译，那么webpack默认读取的将是当前目录下的`webpack.config.js`
```bash
$ pwd
/d/xampp/htdocs/webpack-seed
$ webpack # webpack此时读取的实际上是/d/xampp/htdocs/webpack-seed/webpack.config.js
```
如果你有其它命名的需要或是你有多份配置文件，可以使用`--config`参数传入路径：
```bash
$ webpack --config ./webpackConfig/dev.config.js
```
另外，在CLI执行`webpack`指令时可传入的参数（当然除了`--config`）实际上都可以在配置文件里面直接声明，我强烈建议可以的话尽量都在配置文件里写好，有需要的话写两份配置也好三份也好（反正配置文件间也是可以互相引用的，相同的部分就拆成一个module出来以供读取，最后拼成各种情况下需要的配置就好了）。

## 入口文件配置：entry参数
entry可以是字符串（单入口），可以是数组（多入口），但为了后续发展，请务必使用object，因为object中的key在webpack里相当于此入口的name，既可以后续用来拼生成文件的路径，也可以用来作为此入口的唯一标识。
我推荐的形式是这样的：
```javascript
entry: { // pagesDir是前面准备好的入口文件集合目录的路径
  'alert/index': path.resolve(pagesDir, `./alert/index/page`), 
  'index/login': path.resolve(pagesDir, `./index/login/page`), 
  'index/index': path.resolve(pagesDir, `./index/index/page`),
},
```
对照我的脚手架项目[`webpack-seed`][3]的文件目录结构，就很清楚了：
```bash
├─src # 当前项目的源码
    ├─pages # 各个页面独有的部分，如入口文件、只有该页面使用到的css、模板文件等
    │  ├─alert # 业务模块
    │  │  └─index # 具体页面
    │  ├─index # 业务模块
    │  │  ├─index # 具体页面
    │  │  └─login # 具体页面
```
由于每一个入口文件都相当于entry里的一项，因此这样一项一项地来写实在是有点繁琐，我就稍微写了点代码来拼接这entry：
```javascript
  var pageArr = [
    'index/login',
    'index/index',
    'alert/index',
  ];
  var configEntry = {};
  pageArr.forEach((page) => {
    configEntry[page] = path.resolve(pagesDir, page + '/page');
  });
```

## 输出文件：output参数
output参数告诉webpack以什么方式来生成/输出文件，值得注意的是，与entry不同，output相当于一套规则，所有的入口都必须使用这一套规则，不能针对某一个特定的入口来制定output规则。output参数里有这几个子参数是比较常用的：path、publicPath、filename、chunkFilename，这里先给个[`webpack-seed`][4]中的示例：
```javascript
    output: {
      path: buildDir, // var buildDir = path.resolve(__dirname, './build');
      publicPath: '../../../../build/',
      filename: '[name]/entry.js',    // [name]表示entry每一项中的key，用以批量指定生成后文件的名称
      chunkFilename: '[id].bundle.js',
    },
```

### path
path参数表示生成文件的根目录，需要传入一个**绝对路径**。path参数和后面的filename参数共同组成入口文件的完整路径。

### publicPath
publicPath参数表示的是一个URL路径（指向生成文件的根目录），用于生成css/js/图片/字体文件等资源的路径，以确保网页能正确地加载到这些资源。
publicPath参数跟path参数的区别是：path参数其实是针对本地文件系统的，而publicPath则针对的是浏览器；因此，publicPath既可以是一个相对路径，如示例中的`'../../../../build/'`，也可以是一个绝对路径如`http://www.xxxxx.com/`。一般来说，我还是更推荐相对路径的写法，这样的话整体迁移起来非常方便。那什么时候用绝对路径呢？其实也很简单，当你的html文件跟其它资源放在不同的域名下的时候，就应该用绝对路径了，这种情况非常多见于后端渲染模板的场景。

### filename
filename属性表示的是如何命名生成出来的入口文件，规则有以下三种：

- [name]，指代入口文件的name，也就是上面提到的entry参数的key，因此，我们可以在name里利用`/`，即可达到控制文件目录结构的效果。
- [hash]，指代本次编译的一个hash版本，值得注意的是，只要是在同一次编译过程中生成的文件，这个[hash]的值就是一样的；在缓存的层面来说，相当于一次全量的替换。
- [chunkhash]，指代的是当前chunk的一个hash版本，也就是说，在同一次编译中，每一个chunk的hash都是不一样的；而在两次编译中，如果某个chunk根本没有发生变化，那么该chunk的hash也就不会发生变化。这在缓存的层面上来说，就是把缓存的粒度精细到具体某个chunk，只要chunk不变，该chunk的浏览器缓存就可以继续使用。

下面来说说如何利用filename参数和path参数来设计入口文件的目录结构，如示例中的`path: buildDir, // var buildDir = path.resolve(__dirname, './build');`和`filename: '[name]/entry.js'`，那么对于key为'index/login'的入口文件，生成出来的路径就是`build/index/login/entry.js`了，怎么样，是不是很简单呢？

### chunkFilename
chunkFilename参数与filename参数类似，都是用来定义生成文件的命名方式的，只不过，chunkFilename参数指定的是除入口文件外的chunk（这些chunk通常是由于webpack对代码的优化所形成的，比如因应实际运行的情况来异步加载）的命名。

## 各种Loader配置：module参数
webpack的核心实际上也只能针对js进行打包，那webpack一直号称能够打包任何资源是怎么一回事呢？原来，webpack拥有一个类似于插件的机制，名为`Loader`，通过Loader，webpack能够针对每一种特定的资源做出相应的处理。Loader的种类相当多，有些比较基础的是官方自己开发，而其它则是由webpack社区开源贡献出来的，这里是Loader的List：[list of loaders][5]。
而module正是配置什么资源使用哪个Loader的参数（因为就算是同一种资源，也可能有不同的Loader可以使用，当然不同Loader处理的手段不一样，最后结果也自然就不一样了）。module参数有几个子参数，但是最常用的自然还是`loaders`子参数，这里也仅对loaders子参数进行介绍。

### loaders参数
loaders参数又有几个子参数，先给出一个官方示例：

```javascript
module.loaders: [
  {
    // "test" is commonly used to match the file extension
    test: /\.jsx$/,

    // "include" is commonly used to match the directories
    include: [
      path.resolve(__dirname, "app/src"),
      path.resolve(__dirname, "app/test")
    ],

    // "exclude" should be used to exclude exceptions
    // try to prefer "include" when possible

    // the "loader"
    loader: "babel-loader"
  }
]
```

下面一一对这些子参数进行说明：

- `test`参数用来指示当前配置项针对哪些资源，该值应是一个条件值(condition)。
- `exclude`参数用来剔除掉需要忽略的资源，该值应是一个条件值(condition)。
- `include`参数用来表示本loader配置仅针对哪些目录/文件，该值应是一个条件值(condition)。这个参数跟`test`参数的效果是一样的（官方文档也是这么写的），我也不明白为嘛有俩同样规则的参数，不过我们姑且可以自己来划分这两者的用途：`test`参数用来指示文件名（包括文件后缀），而`include`参数则用来指示目录；注意同时使用这两者的时候，实际上是`and`的关系。
- `loader`/`loaders`参数，用来指示用哪个/哪些loader来处理目标资源，这俩货表达的其实是一个意思，只是写法不一样，我个人推荐用`loader`写成一行，多个loader间使用`!`分割，这种形式类似于`管道`的概念，又或者说是`函数式编程`。形如`loader: 'css?!postcss!less'`，可以很明显地看出，目标资源先经less-loader处理过后将结果交给postcss-loader作进一步处理，然后最后再交给css-loader。

条件值(condition)可以是一个字符串（某个资源的文件系统绝对路径），可以是一个函数（官方文档里是有这么写，但既没有示例也没有说明，我也是醉了），可以是一个正则表达式（用来匹配资源的路径，最常用，强烈推荐！），最后，还可以是一个数组，数组的元素可以为上述三种类型，元素之间为与关系（既必须同时满足数组里的所有条件）。需要注意的是，loader是可以接受参数的，方式类似于URL参数，形如'css?minimize&-autoprefixer'，具体每个loader接受什么参数请参考loader本身的文档（一般也就只能在github里看了）。

## 添加额外功能：plugins参数
这plugins参数相当于一个插槽位（类型是数组），你可以先按某个plugin要求的方式初始化好了以后，把初始化后的实例丢到这里来。