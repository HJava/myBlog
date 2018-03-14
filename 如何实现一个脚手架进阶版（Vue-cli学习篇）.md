# 前言

在之前一篇博客介绍了关于Node脚手架的一些基础的知识，这篇博客是在之前的基础上针对在Node中开发脚手架中遇到的问题，如：

- 终端样式、交互问题
- 文件处理问题

通过对Vue-cli 2.9.2的源码进行分析，解决相关问题。

如果没有看过之前一篇博客的，或者对Node.js的脚手架没有了解过的同学，推荐先看上一篇：[如何实现一个简单的Node.js脚手架](https://juejin.im/post/5a88dee46fb9a0635d0c2540)。

# 正文

## 终端

### 怎么自定义终端的样式

[chalk](https://github.com/chalk/chalk)是一个日志的样式库，可以在终端上面调整日志的样式。

以下的代码：

```javascript
const chalk = require('chalk');

console.log(chalk.red('hello world'));
```

转换为具体样式为：

![](https://segmentfault.com/img/bV2Oxf)

通过说明文档，我们可以知道，chalk可以支持文字颜色和背景颜色。

![clipboard.png](https://segmentfault.com/img/bV2Otc)

### 怎么实现终端中的Loading图

[ora](https://github.com/sindresorhus/ora)可以在终端实现Loading的效果，可以在与用户进行交互后使用。

从官网的实例来看，我们可以实现以下的效果：

![](https://camo.githubusercontent.com/5668f3a74ede2534f047a1cdda3a6a659939aa3d/68747470733a2f2f7261776769742e636f6d2f73696e647265736f726875732f6f72612f6d61737465722f73637265656e73686f742e737667)

### 怎么在终端中和用户进行交互

在终端中和用户进行交互，获取用户输入是一个很常见的需求。

[Inquirer](https://github.com/SBoudrias/Inquirer.js)这个库能够通过终端让我们和用户进行一些交互，比如下面的例子：

```javascript
var inquirer = require('inquirer');
inquirer.prompt([{type: 'confirm', name: 'OK', message: 'Are you OK?', default: false}]).then(answers => {
    console.log(answers);
});
```

得到的结果内容为：

![](https://segmentfault.com/img/bV2OIi)

## 文件相关

### 怎么能够方便下载有目录结构的模板文件

最常见的文件模板下载，都是通过将文件上传到CDN中，然后再通过某个特定的格式下载到页面上。

但是，如果你想要通过比较优雅的方式来进行文件下载，可以通过[download-git-repo](https://github.com/flipxfx/download-git-repo)来下载你再Git上面已经准备好的模板，这样就能够在下载的过程中保证文件目录和顺序，比之前自己创建和检测文件夹会简便很多。

### 怎么对下载的文件进行处理

当你提供的模板不仅仅是一个纯粹的文件，而是可以通过某些参数进行编译，得到不同的目标文件时，你可以通过[metalsmith](https://github.com/segmentio/metalsmith)来对文件进行操作。

它是一个用来构建静态网站的类库，也能够用来对文件进行处理。

你可以通过编写一些处理的回调函数来对你的模板文件进行处理。

### 怎么编译模板语言

如果你想要一套现成的模板编译工具，可以使用现成的如[Handlebars](https://github.com/daaain/Handlebars)。

他能够像后端模板语言一样，直接针对类HTML文件进行处理，我们可以看下官方的例子。

```html
<div class="entry">
  <h1>{{title}}</h1>
  <div class="body">
    {{body}}
  </div>
</div>
```

针对上述模板，在编译时填入`title`和`body`两个字段，即可得到完整的HTML片段。

在`Vue-cli`中，使用了模板引擎合并库[consolidate.js](https://github.com/tj/consolidate.js/),通过这个库的`render`方法来编写metalsmith的处理函数，从而在渲染的过程中对模板进行处理，具体代码如下：

```javascript
exports.handlebars.render = function(str, options, fn) {
  return promisify(fn, function(fn) {
    var engine = requires.handlebars || (requires.handlebars = require('handlebars'));
    try {
      for (var partial in options.partials) {
        engine.registerPartial(partial, options.partials[partial]);
      }
      for (var helper in options.helpers) {
        engine.registerHelper(helper, options.helpers[helper]);
      }
      var tmpl = cache(options) || cache(options, engine.compile(str, options));
      fn(null, tmpl(options));
    } catch (err) {
      fn(err);
    }
  });
};
```

因此，他可以利用已经安装好的handlebar模板引擎来注册helper，从而进行模板的处理。

# 总结

通过对`Vue-cli`源码的简单学习，我们可以发现在日常中需要处理的与用户交互、文件处理编译等需求也有了一个比较好的解决方案。

当然，上面提供的方案不是唯一的，仅提供给大家进行一个参考和快速实现。大家也可以通过一些其他的方法来实现相同的功能。

有任何问题欢迎进行交流。