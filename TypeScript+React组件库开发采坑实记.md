# 概述

本文为配置TypeScript+Webpack+React，开发UI组件库时遇到的坑以及相对应的解决方案记录，适合相关同学进行查阅解决问题。

本文主要内容为：

- `tsconfig.js`配置中遇到的问题
- 选择TypeScript Loader遇到的问题
- Webpack遇到的问题

此三类配置和选择会同时导致某一类问题，因此这三类不作为分类标准，仅作为读者思考的方向，我们下面会根据具体的问题和错误以及对应的解决方案来进行说明。

本文只需略读，在遇到相对应问题时，能够知道此处有提供相对应解决方案可以供尝试即可。

# 开发中遇到的问题

## 在项目中使用Webpack对应的alias

在使用TypeScript时，如果需要使用alias功能，有以下两种方法。相关解决方案参考[Stack Overflow](https://stackoverflow.com/questions/40443806/webpack-resolve-alias-does-not-work-with-typescript)上面的回答，不过此回答仍然有部分内容没有考虑到，我们将在下面介绍时进行说明。

### 使用ts-loader

使用ts-loader作为loader来编译TypeScript时，你需要在TypeScript中配置`baseUrl`和`paths`，具体配置可以参考下面示例：

```json
{
    "baseUrl": ".",
    "paths": {
        "Src/*": [
            "src/*"
        ],
        "Interfaces/*": [
            "src/interfaces/*"
        ],
        "Components/*": [
            "src/components/*"
        ],
        "Utils/*": [
            "src/utils/*"
        ]
    }
}
```

**注：在paths中，绝对路径不允许以@开头，如：将@utils的绝对路径解析到src/utils中。**

在`tsconfig.json`配置完成后，使用[TsConfigPathsPlugin](https://github.com/dividab/tsconfig-paths-webpack-plugin)插件来读取相关配置文件到`webpack.config.js`中，官方示例配置如下：

```javascript
const TsconfigPathsPlugin = require('tsconfig-paths-webpack-plugin');

module.exports = {
  ...
  resolve: {
    plugins: [new TsconfigPathsPlugin({ /*configFile: "./path/to/tsconfig.json" */ })]
  }
  ...
}
```

通过这种方法，使用ts-loader就能够使用alias功能了。

### 使用awesome-typescript-loader

**推荐使用此loader来编译TypeScript，速度相较于ts-loader来说，肉眼可见的快。**

在使用awesome-typescript-loader时，在Stack Overflow的回答中说这个插件会将`tsconfig.json`中的配置文件自动添加到Webpack中。经过实践发现，当绝对路径最终引用的文件是一个Interface时，只需要在`tsconfig.js`中进行指定即可找到相对应文件。

当绝对路径最终引用的文件是一个Function时，如果不指定webpack alias字段，则会出现相关错误，如`Module not found: Error: Can't resolve 'Utils/handle-url'`。需要解决此问题，只需要在webpack alias中添加相同配置的alias即可解决。

## Webpack支持ts/tsx后缀

需要让Webpack支持ts/tsx后缀，需要在`extension`字段中添加相对应的值，具体如下：

```json
{
    "extensions": [".ts", ".tsx"]
}
```

**注：需要增加的字段都需要带上'.'，否则Webpack仍然不识别。**

## 无法解析tsx文件

出现错误`jsx  You may need an appropriate loader to handle this file type.`。

需要解决此问题，需要增加`tsconfig.json`中的`jsx`相对应的值，具体如下：

```json
{
    "jsx": "react"
}
```

## 无法找到@types/csstypes

在编译时，出现无法找到@types/csstypes错误，具体错误内容为`Cannot find module 'csstype'.`。

需要解决此问题，需要增加`tsconfig.json`中的`jsx`相对应的值，具体如下：

```json
{
    "moduleResolution": "node"
}
```

## 元素隐含地具有“任何”类型，因为类型“窗口”没有索引签名？

将部分JavaScript文件改成TypeScript文件时，会出现此错误，我遇到的问题是因为：定义的全局对象没有确定格式。

需要解决此问题，需要增加指定的Interface，格式如下：

```typscript
Interface ObjInterface {
    [key: string]:string
}
```

# 总结

本文是我在工作中配置TypeScript+Webpack+React开发相关脚手架时遇见的问题，大家只需要阅读相关目录，对相对应的问题有所了解。在遇到相对应的问题时，再搜索进行解决即可。








