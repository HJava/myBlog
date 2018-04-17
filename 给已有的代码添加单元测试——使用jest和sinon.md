# 概述

在日常的功能开发中，我们的代码测试都依赖于自己或者QA进行测试。这些操作不仅费时费力，而且还依赖开发者自身的驱动。在开发一些第三方依赖的库时，我们也没有办法给第三方提供完整的代码质量报告。

现在，我们可以使用单元测试来提高自己的代码质量。下面，我将自己在使用Jest和Sinon.js配置和编写单元测试中的收获的经验和踩到的坑进行总结，根据从零开始配置和编写单元测试这一条线来进行分享。

通过本文，你可以解决以下问题：

- Jest与Sinon.js是什么？
- 如何配置Jest与Sinon.js，从而编写单元测试？
- 如何使用nyc来统计单元测试覆盖率？
- 如何解决进行单元测试中遇到的常见问题？ 

# Jest与Sinon.js是什么

[Jest](https://facebook.github.io/jest/)是FaceBook推出的一个针对JavaScript进行单元测试的库，它提供了断言、函数模拟等API来对你自己编写的业务逻辑代码进行测试后。

[Sinon.js](http://sinonjs.org/)是一个用来做独立测试和模拟的JavaScript库。它在单元测试的编写中通常用来模拟HTTP等相关请求。

## 为什么没有用其他的单元测试框架

在最开始的框架选择中，我先尝试了能够并行测试，大大提高单元测试速度的ava框架。它能满足日常的普通需求如utils工具集的测试，也能够配置Sinon.js来进行HTTP模拟测试。

但是在处理webpack alias的问题上，通过官方[issue](https://github.com/avajs/ava/issues/1011)中的极其复杂的配置也没有能够解决出现`Cannot find module`的问题；而在Jest中，可以很方便的通过一些简单配置，就能够识别在文件中使用的webpack alias，相关的具体方法将会在后面章节进行具体描述。

而对于其他的测试框架如：[Mocha](https://mochajs.org/)或者[Chai](http://www.chaijs.com/)等，没有进行具体的了解，因此在这里不多做评价。

# 如何配置Jest与Sinon.js，从而编写单元测试？

## Jest配置

### 安装依赖包

需要使用Jest，首先你需要进行安装，执行以下命令:

```
npm install jest -D
```

如果你的项目中存在`.babelrc`文件（使用了babel 6）时，不论你测试的代码是否通过babel进行编译，你都需要安装额外的几个包：

```
npm install babel-jest babel-core regenerator-runtime -D
```

如果你使用的是babel 7，则需要安装下面几个包：

```
npm install babel-jest 'babel-core@^7.0.0-0' @babel/core regenerator-runtime -D
```

### package.json文件配置

在安装完成依赖包以后，你还需要在`package.json`文件中配置如下字段：

```json
{
  "jest": {
    "transform": {}
  }
}
```

`.babelrc`文件只需要保存之前的配置，不需要做任何修改即可生效。

## Sinon.js使用

### 依赖包安装

安装配置完了Jest，让我们来看下Sinon.js。需要使用Sinon.js，我们首先需要进行安装：

```
npm install sinon -D
```

配置完成后，需要在使用的地方进行引入，如下所示：

```javascript
const sinon = require('sinon');
```

## 编写单元测试



# 如何使用nyc来统计单元测试覆盖率？



# 如何解决进行单元测试中遇到的常见问题？ 



# 总结



# 附录



