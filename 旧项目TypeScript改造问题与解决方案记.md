# 概述

由于本次改造的项目为一个通过NPM进行发布的基础服务包，因此本次采用TypeScript进行改造的目标是移除Babel全家桶，减小包体积，同时增加强类型约束从而避免今后开发时可能的问题。

本次改造使用的是TypeScript v2.9.2，采用Webpack v4.16.0进行打包编译。开发工具使用的是VSCode，使用中文语言包。预期目标是直接将TypeScript代码通过loader直接编译为ES5的代码。

本文中涉及的问题有部分是TypeScript配置和使用的问题，也有部分是VSCode本身配置相关问题。

# 改造问题记录与分析

## VSCode相关

### “无法找到相关模块”报错

在项目中，如果我们使用了webpack.alias，可能会提示找不到模块。

具体错误如下：

```
终端编译报错：TS2307: Cannot find module '_utils/index'.
编辑器报错：[ts]找不到模块“_utils/index”。
```

这是由于编辑器无法读取对应的别名信息导致的。

此时我们需要检查对应的模块是否存在。如果确认模块存在，且终端编译编译时不报错，而只是编辑器报错，则是因为编辑器无法读取webpack配置，我们需要增加另外的配置。

解决方法：除了配置webpack.alias，还需要配置相对应的`tsconfig.json`，具体配置如下所示：

```json
"compilerOptions": {
    "baseUrl": ".",
    "paths": {
        "_util/*": [
            "src/core/utils/*"
        ]
    }
}
```

**注：如果配置了`tsconfig.json`以后还是报错的话，需要重启下VSCode，猜测是由于VSCode只在项目加载时读取相关配置信息。在JavaScript项目中的`jsconfig.json`同理。**

## TypeScript相关

### 对象属性赋值报错

在JavaScript中，我们经常会声明一个空对象，然后再给这个属性进行赋值。但是这个操作放在TypeScript中是会发生报错的:

```typescript
let a = {};

a.b = 1;
// 终端编译报错：TS2339: Property 'b' does not exist on type '{}'.
// 编辑器报错：[ts] 类型“{}”上不存在属性“b”。
```

这是因为TypeScript不允许增加没有声明的属性。

因此，我们有两个办法来解决这个报错：

1. 在对象中增加属性定义（推荐）。具体方式为：`let a = {b: void 0};。`这个方法能够从根本上解决当前问题，也能够避免对象被随意赋值的问题。

2. 在对象中添加类型定义（推荐）。具体方式为如下：

   ```typescript
   interface obj {
       [propName: string]: any
   };
   let a: obj = {};
   
   a.a = 1;
   ```

   这样也能够避免报错问题，并且不引入全对象any情况。

3. 给`a`对象增加any属性（应急）。具体方式为：`let a: any = {};`。这个方法能够让TypeScript类型检查时忽略这个对象，从而编译通过不报错。这个方法适用于大量旧代码改造的情况。

### Window对象属性赋值报错

与上一个情况类似，我们给一个对象中赋值一个不存在的属性，会出现编辑器和编译报错：

```typescript
window.a = 1;
// 终端编译报错：TS2339: Property 'a' does not exist on type 'Window'.
// 编辑器报错：[ts] 类型“Window”上不存在属性“a”。
```

这也是因为TypeScript不允许增加没有声明的属性导致的。

由于我们没有办法声明windows属性的值（或者说很困难），因此我们需要通过下面这一种方式来解决：

1. 我们在windows使用时增加一个类型转换，即`(window as any).a = 1;`。这样就能够保证编辑器和编译时不会出错。不过该方法只建议用于旧项目改造，我们还是要尽量避免在window对象上面增加属性，应该通过一个全局的数据管理器来进行数据存取。

### ES2015 Object新增的原型链上的方法报错

在项目中，使用到了一些Object原型链上面的一些ES2015新增的方法，如`Object.assign`和`Object.values`等，此时编译会失败，同时VSCode会提示报错：

```
终端编译报错：TS2339: Property 'assign' does not exist on type 'ObjectConstructor'.
编辑器报错：[ts] 类型“ObjectConstructor”上不存在属性“assign”。
```

这是由于我们在`tsconfig.json`中指定的`target`是ES5，而TypeScript并没有相关的polyfill，因此我们无法使用ES2015中新增的方法。

通过以上分析，我们可以使用如下方法解决：

1. 可以使用lodash工具集中的相关方法，安装时需要安装`lodash.assign`和`@types/lodash.assign`。并且`lodash.assign`是一个CMD规范的包，需要通过`import _assign = require('lodash.assing');`方式引入。

2. 我们可以使用rest写法，例如`let a = {...b};`，也能够达到一级浅拷贝的效果，具体效果如下：

   ![image.png](https://file.neixin.cn/pan/im/2/image/Aj_NK_JZqHTbYZd3_si-whIcSGrXtLTOz0i9OoUuo5w3V4MjijWPj3SCKW4BKHJRTg?filename=image.png)

### ES2015新增的数据结构Map初始化报错

将ES2015的代码改造成为TypeScript代码时，如果你使用了ES2015新增的Map类型，那在编辑器还是终端编译中编译时都会报错：

```
终端编译报错：TS2693: 'Map' only refers to a type, but is being used as a value here.
编辑器报错报错：[ts] “Map”仅表示类型，但在此处却作为值使用。
```

这是由于TypeScript并没有提供相关的数据类型，也没有对应的polyfill。

因此，我们解决这个问题的思路有三种：

1. 将`tsconfig.json`配置中的`target`属性改为`es6`，即输出符合ES2015规范的代码。因为ES2015存在全局的Promise对象，因此编译和编辑器都不会报错。该方法优点为配置简单，无需改动代码，缺点为需要高级浏览器的支持或者Babel全家桶的支持。
2. 舍弃Map类型，改用Object进行替代。这种改造比较费时费力，适用于工作量较小和不愿意引入其他文件的场景。
3. 自行实现或者安装一个Map包。这种方法改造成本较小，缺点就是会引入额外的代码或者包，并且代码效率无法保证。例如`ts-map`和`typescript-map`，这两个包的查找效率都是o(n)，低于原生类型的Map。因此推荐自己使用Object实现一个简单的Map，具体实现方式可以去网上找相关的Map原理分析与实践（大致原理为使用多个Object，存储不同类型元素时使用不同容器，避免类型转换问题）。

### ES2015新增的Promise使用报错

将ES2015的代码改造成为TypeScript代码时，如果你使用了ES2015的新增的Promise类型，那在编辑器还是终端编译编译时都会报错：

```
终端编译报错： TS2693: 'Promise' only refers to a type, but is being used as a value here.
编辑器报错：[ts] “Promise”仅表示类型，但在此处却作为值使用。
```

这是由于TypeScript并没有提供Promise数据类型，也没有对应的polyfill。

因此，我们解决这个问题的思路仍然有三种：

1. 将`tsconfig.json`配置文件配置中的`target`属性改为`es6`，即输出符合ES2015规范的代码。因为ES2015存在全局的Promise对象，因此编译和编辑器都不会报错。该方法优点为配置简单，无需改动代码，缺点为需要高级浏览器的支持或者Babel全家桶的支持。

2. 引入一个Promise库，如bluebird等比较知名的Promise库。在安装bluebird时需要同时安装@types/bluebird声明文件。缺点就是引入的Promise库较大，而且如果你的库作为一个基础库时，可能会与其他的调用方的Promise库产生冲突。

3. 在`tsconfig.json`配置文件中增加lib。此方法的原理是让TypeScript编译时引用外部的Promise对象，因此在编译时不会报错。此方式优点是不会引入任何其他代码，但是缺点是一定要保证在引用此库的前提下，一定存在Promise对象。具体配置如下：

   ```
   "compilerOptions": {
   	"lib": ["es2015.promise"]
   }
   ```

### SetTimeout使用报错

将ES2015代码改造成TypeScript代码时，如果使用了setTimeout和setInterval函数时，可能会出现无法找到该函数的报错：

```
终端编译报错：TS2304: Cannot find name 'setTimeout'.
编辑器报错：[ts] 找不到名称“setTimeout”。
```

这是由于编辑器和编译时不知道当前代码运行环境导致的。

因此，我们解决这个问题的思路有两种：

1. 在`tsconfig.json`配置文件中增加lib。让TypeScript能够知道当前的代码容器。具体示例如下：

   ```
   "compilerOptions": {
   	"lib": ["dom"]
   }
   ```

2. 安装`@types/node`。该方法适用于node环境下或者采用webpack打包时可以引入node代码。该方法直接通过`npm install @types/node`即可安装完成，解决报错问题。

### 模块引用和导出报错

在ES2015的代码中，我们可以通过`@babel/plugin-proposal-export-default-from`插件来直接导出引入的文件，具体示例如下：

```javascript
export Session from './session'; // 报错
export * from '_models/read-item'; // 不报错
```

而在TypeScript中，这种写法是会报错的：

```
终端编译报错：TS1128: Declaration or statement expected.
编辑器报错：[ts] 应为声明或语句。
```

这是由于两者的模块语法不一样导致的。

因此，我们解决这个问题只需要用下面这一种方法：

1. 将上面的`export from`的语法稍加调整来适配TypeScript语法。具体改造如下：

   ```typescript
   export {default as Session} from '_models/session'; //调整后不报错
   export * from '_models/read-item';// 之前不报错不需要调整
   ```

### 泛型定义

我们在项目中经常会遇到这种情况，我们需要保证传入的属性类型的同时，还需要保证其与某个函数的参数一致，如：

```typescript
interface props {
    value: number | string, 
    onChange: (v: string | number) => void // 参数类型值需要与value一致
}
```

为了解决这个问题，我们需要用到泛型定义：

```typescript
interface Props<T extends string | number> {
    value: T,
    onChange: (v: T) => void
}
```

此时，当value的类型确定时，参数的类型也就变得和value一样确定了。

# 总结

在做项目TypeScript改造的过程中，遇到了不少大大小小的坑。很多问题在网上都没有解决方案或者没有说明白具体的解决步骤，因此希望通过这一篇文章来帮助大家在进行TypeScript迁移时避免在我踩过的坑上再浪费时间。
