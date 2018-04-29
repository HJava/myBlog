# 概述

在我们进行日常的项目开发的过程中，我们经常会遇到使用一些通用的UI组件库如BootStrap、Ant Design等。作为成熟的UI组件库，它能够提供提供一整套UI组件用来满足使用需求，能大大减少开发成本。

在使用了他人提供的组件库后，我自然就会有兴趣去了解一下别人开发的组件库到底是如何设计的，如何进行相关的组件封装。本文以Ant Design为例，让我们来了解一下目前较为有名的UI组件库是如何设计与实现的？同时，我们又能够有哪些经验可以借鉴？

阅读本文，你最好有如下的基础知识来帮助你理解本文内容：

- 对React相关开发有一定的了解，对React的组件有一定的认知。
- 对JavaScript有一定的了解，最好能够了解TypeScript（不了解也能够理解文章内容）。
- [选]对单元测试有一定的了解（不了解可以先阅读我的前一篇文章[提高代码质量——使用Jest和Sinon给已有的代码添加单元测试](https://juejin.im/post/5ad75f16f265da50602c597b)）。

PS: 博客写一半，手受伤骨折了囧囧囧，后面大部分文字是通过语音输入转换的，如果有什么错误或者逻辑不清晰，欢迎在评论中指正。

# 如何实现单个的React UI组件

首先，需要了解Ant Design提供的组件，我们先来看下单个组件是如何实现的。

## 目录结构

需要了解一个组件的内容，我们应该先从目录接口开始。我们以avatar头像组件为例，目录结构如下图所示：

![](https://file.neixin.cn/pan/im/2/image/AoJzQSUf9obBDtHYoweEELpd80IjDDvx-cO9NXWcNOAeIbddzOshSMxaFbOK4jrU9A?filename=image.png)

我们一个一个来看一下：

- index.tsx，UI组件源文件，即TSX（TypeScript+JSX），包含整个组件的内容和逻辑。
- index.zh-CN.md，组件使用说明文档
- style，UI组件样式文件，包含当前UI组件的相关样式
- __tests__，UI组件测试文件，包含当前UI组件相关的单元测试，使用Jest单元测试框架。
- demo，用来进行展示和用法示例说明的文档

介绍完了目录结构，我们来看下这个插件的具体内容。

## TSX文件

我们首先来看一下这个插件的TSX文件。这个文件包含了插件的结构和功能。如代码示例示例所示：

```typescript
export interface AvatarProps {
  /** Shape of avatar, options:`circle`, `square` */
  shape?: 'circle' | 'square';
  /** Size of avatar, options:`large`, `small`, `default` */
  size?: 'large' | 'small' | 'default';
  /** Src of image avatar */
  src?: string;
  /** Type of the Icon to be used in avatar */
  icon?: string;
  style?: React.CSSProperties;
  prefixCls?: string;
  className?: string;
  children?: any;
}

export interface AvatarState {
  scale: number;
  isImgExist: boolean;
}
```

我们先看一下声明文件。在你TypeScript声明中我们可以看到：它通过interface定义了props和state两个属性的值。这样可以明确界定传入的属性和内部的属性类型，在代码规范和质量中也能够有一个保证。

下面让我们来看一下具体的组件类。具体示例如下：

```typescript
export default class Avatar extends React.Component<AvatarProps, AvatarState> {
    render() {
        const {
          prefixCls, shape, size, src, icon, className, ...others,
        } = this.props;
    
        const sizeCls = classNames({
          [`${prefixCls}-lg`]: size === 'large',
          [`${prefixCls}-sm`]: size === 'small',
        });
    
        const classString = classNames(prefixCls, className, sizeCls, {
          [`${prefixCls}-${shape}`]: shape,
          [`${prefixCls}-image`]: src && this.state.isImgExist,
          [`${prefixCls}-icon`]: icon,
        });
    
        let children = this.props.children;
        if (src && this.state.isImgExist) {
          children = (
            <img
              src={src}
              onError={this.handleImgLoadError}
            />
          );
        } else if (icon) {
          children = <Icon type={icon} />;
        } else {
          const childrenNode = this.avatarChildren;
          if (childrenNode || this.state.scale !== 1) {
            const childrenStyle: React.CSSProperties = {
              msTransform: `scale(${this.state.scale})`,
              WebkitTransform: `scale(${this.state.scale})`,
              transform: `scale(${this.state.scale})`,
              position: 'absolute',
              display: 'inline-block',
              left: `calc(50% - ${Math.round(childrenNode.offsetWidth / 2)}px)`,
            };
            children = (
              <span
                className={`${prefixCls}-string`}
                ref={span => this.avatarChildren = span}
                style={childrenStyle}
              >
                {children}
              </span>
            );
          } else {
            children = (
              <span
                className={`${prefixCls}-string`}
                ref={span => this.avatarChildren = span}
              >
                {children}
              </span>
            );
          }
        }
    return (
      <span {...others} className={classString}>
        {children}
      </span>
    );
  }
}
```

从上面的示例代码中我们可以看到，这是一个很常规的React的组件类。它通过传入的属性来判断应该选择哪种方式渲染头像，然后完成组件的渲染过程。同时，在组件渲染完成后，这个组件会根据参数来调整相关的图片大小用于适配。

这样，一个简单的UI组件就已经基本满足相关的功能了。

接下来，让我们来看下样式相关的文件。

### Style文件

还是以Avatar组件为例，具体的样式代码这里就不举例了，只像大家介绍下：Ant Design的样式是通过Less语言来完成的，并且通过TSX文件来进行引入。

### Test文件

Test文件通过[enzyme](https://github.com/airbnb/enzyme)来对React组件进行测试。我们简单介绍以下enzyme，这是一个对React组件进行测试的JavaScript框架，能够提供mount等相关API接口来对组件选人和相关的数据更新操作进行测试。

我们选取一部分测试示例如下：

```javascript
import React from 'react';
import { mount } from 'enzyme';
import Avatar from '..';

describe('Avatar Render', () => {
  it('Render long string correctly', () => {
    const wrapper = mount(<Avatar>TestString</Avatar>);
    const children = wrapper.find('.ant-avatar-string');
    expect(children.length).toBe(1);
  });

  it('should render fallback string correctly', () => {
    const div = global.document.createElement('div');
    global.document.body.appendChild(div);

    const wrapper = mount(<Avatar src="http://error.url">Fallback</Avatar>, { attachTo: div });
    wrapper.instance().setScale = jest.fn(() => wrapper.instance().setState({ scale: 0.5 }));
    wrapper.setState({ isImgExist: false });

    const children = wrapper.find('.ant-avatar-string');
    expect(children.length).toBe(1);
    expect(children.text()).toBe('Fallback');
    expect(wrapper.instance().setScale).toBeCalled();
    expect(div.querySelector('.ant-avatar-string').style.transform).toBe('scale(0.5)');

    wrapper.detach();
    global.document.body.removeChild(div);
  });
});
```

通过上面的示例我们可以知道，enzyme能够根据Avatar组件渲染后的数据来对组件进行测试。

## UI组件库到底是如何实现以及与使用者交互的

其实UI组件与我们自己开发的React组件没有什么太大的区别，只是一个提供了部分UI和功能的第三方组件而已。想明白了这点，我们就能知道，我们开发的组件与第三方UI组件的交互就是通过Props的方式。

以上面的Avatar组件为例，我们给组件传递`sharp`, `size`，`src`等字段，Avatar组件收到相关数据后，在内部进行相关的处理，最终返回一个React组件。具体示例如下：

```typescript
import Avatar from './avatar/';

class Container extends React.Component {
    render() {
        return (
            <div>
                <Avatar sharp="circle"  size="large"  src="https://www.baidu.com">Avatar!!</Avatar>
            </div>
        );
    }
}
```

如果我们需要引入相关样式文件，我们则需要引入编译后的css文件，具体方式如下：

```css
@import '~antd/dist/antd.css';
```


## 通过Ant Design，我们在开发UI组件时学到了什么

通过对UI组件库源码的阅读，我得到了如下的一些经验。

### 结构清晰

每一个UI组件都是一个完整的模块，都应该有自己独立的目录结构；同时所有的UI组件都是属于同一类，因此所有的UI组件的目录结构应该相似。

Ant Design中每一个UI组件的目录结构都如前几章中所述。拥有一个清晰的目录结构能够方便我们进行代码管理，同时也可以使用脚本做一些自动化的处理。比如Ant Design就通过脚本来对所有`components/**/style`文件夹中的less文件进行合并编译。

### 组件分离

每个UI组件应该都是可以独立被引用的，而且也应该优先使用“独立引用”的方式。

Ant Design的每一个组件都可以被独立引用，引用方式如下：

```javacript
import Button from 'antd/lib/button';
```

我们在使用第三方UI组件库时，通常不会使用到上面所有的组件，而是经常使用到部分组件。因此我们在设计UI组件库时，处于文件大小的考虑，我们也应该保证每个UI组件都互不依赖（同一层级的组件，排除本身业务上就有依赖关系的组件），做到不使用的组件不引入，减小业务方文件大小。

### 测试覆盖

每一个UI组件都是独立的，因此我们需要为每一个独立的组件进行测试覆盖。

Ant Design中通过Jest和上文提到的enzyme来对每一个组件进行测试，从而保证UI组件的代码质量。


# 总结

总体上来说，Ant Design相关的源代码简单易懂，结构也很清晰，非常容易阅读。如果你对React开发有一定的了解，但是不知道如何进行组件的封装，或者想了解当前主流的组件库是如何实现的，推荐可以阅读一下相关源码。你能够从中了解到我们如何对UI组件进行切割和封装。

当然，Ant Design不仅仅是一个UI组件库，而是一整套UI规范，我们今天分享的只是这套规范在React上面的实现。如果对相关的UI规范有兴趣的同学，可以去Ant Design的[官网](http://ant.design/index-cn)进行了解。



