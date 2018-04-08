# 概述

本文通过对chrome插件的各个部分进行快速的介绍，从而让大家了解插件各个部分的关系，并且知道如何将其进行组装成一个完整的chrome插件。

由于chrome官方文档中对于如何从零开发一个chrome扩展应用没有一套完整的流程，同时官方的API文档对于初学者也不是那么友好，因此本文将通过一个初学者的视角来讲解如何从零开始快速了解和开发一个chrome插件。

本文的目标群体：已经了解或使用过chrome扩展应用，但是自己不知道如何开发一个chrome扩展应用的工程师。如果有具体的chrome扩展应用开发经验的同学，本篇文章可能太过简单，并不适合你。

本文的主要内容如下：

- chrome扩展应用模块功能介绍
- chrome扩展应用模块开发介绍

本文的内容不包括chrome扩展应用开发时提供的各个API功能详解，有需求的同学可以自行查看[官方API文档](https://developer.chrome.com/extensions/api_index)。

# chrome扩展应用模块功能介绍

chrome扩展应用由很多部分组成，其中主要模块为：

- Manifest File


- Background Pages
- Content Script
- Options

为了避免由于翻译原因导致的问题，因此在下文中对相关模块的称呼一律采用上面的英文。下面，我们先简单来了解下这些模块具体是什么作用。

## Background Pages

> A common need for extensions is to have a single long-running script to manage some task or state. Background pages to the rescue.

从官方的介绍我们可以知道，`Background Pages`的作用就是在浏览器运行时，会长时间执行的脚本。只要浏览器处于打开状态，在`Background Pages`中的脚本就会在后台执行。

## Content Script

> Content scripts are JavaScript files that run in the context of web pages. By using the standard [Document Object Model](http://www.w3.org/TR/DOM-Level-2-HTML/) (DOM), they can read details of the web pages the browser visits, or make changes to them.

从上面官方的介绍我们可以知道，`Content Script`其实就是我们需要写的将会在我们希望的目标页面中执行的脚本文件。每次目标页面刷新时，这部分脚本也会重新加载执行。

## Options

> To allow users to customize the behavior of your extension, you may wish to provide an options page.

从官方的介绍我们可以了解，`Options`部分就是我们对于扩展的管理功能。我们能够通过一个模块来对chrome扩展应用的设置和数据进行处理。

# chrome扩展应用模块开发介绍

首先，让我们先确定我们插件需要完成的一个功能，这样我们就能够有一个目标示例来进行介绍。

以我自己开发的表情插件为例，它必须具备以下几项功能：

- 能够收集任何网页的图片作为表情
- 能够在插件中管理已有表情
- 能够在特定页面中将表情发送出去

我们将上面的功能抽象一下，就能够得到如下的结果：

- 能够收集保存任何网页的图片作为表情（长时间执行脚本监听用户操作）
- 能够在特定页面中将保存的表情发送出去（在目标页面中使用脚本与页面进行交互）
- 能够在插件中管理已有表情（插件管理相关功能）

因此，需要完成上述功能，我们就需要用到上面我们提到的功能模块。下面，让我们按照模块来看下，我们应该如何实现这些功能。

## 配置文件（Manifest File）

首先，在进行具体的功能开发时，我们需要来看下我们的项目配置文件。这个配置文件在整个chrome扩展应用中非常重要，包含了项目的属性、配置、权限和资源信息。

```json
{
  "manifest_version": 2,
  "name": "大象表情收藏",
  "description": "大象表情收藏（非官方）",
  "version": "4.15.1",
  "default_locale": "zh_CN",
  "icons": {
    "16": "img/icon16.png",
    "48": "img/icon48.png",
    "128": "img/icon128.png"
  },
  "background": {
    "scripts": [
      "js/background.js"
    ],
    "persistent": false
  },
  "permissions": [
    "<all_urls>",
    "storage",
    "contextMenus"
  ],
  "content_scripts": [
    {
      "css": [
        "js/main.css"
      ],
      "js": [
        "js/favor.js"
      ],
      "matches": [
        "*://x.sankuai.com/*"
      ],
      "run_at": "document_end"
    }
  ],
  "options_page": "options.html",
  "web_accessible_resources": [
    "img/favorite.png",
    "img/left.svg",
    "img/right.svg",
    "img/icon128.png",
    "img/plane.svg",
    "options.html"
  ]
}
```

这是我开发的大象表情插件的manifest配置文件，我们通过这个配置文件来看下相关的属性字段。

| 属性名称                 | 属性含义               | 备注                                                         |
| ------------------------ | ---------------------- | ------------------------------------------------------------ |
| manifest_version         | manifest文件版本       |                                                              |
| name                     | 项目名称               | 发布到商店时的名称。                                         |
| description              | 项目简介               | 发布到商店时的简介。                                         |
| version                  | 项目版本               | 发布到商店时需要每次递增。                                   |
| default_locale           | 默认的locale目录       | 具体见[此处](https://developer.chrome.com/apps/manifest/default_locale)。 |
| icons                    | 扩展应用图标           | 需要提供16x16，48x48，128x128三种尺寸。                      |
| background               | Background Pages文件   |                                                              |
| permissions              | 扩展应用所需权限       | 权限列表见[此处](https://developer.chrome.com/apps/declare_permissions)。申请权限后，可以使用chrome对象来进行访问该权限提供的API接口。我所开发的扩展应用主要是使用到了右键菜单和存储权限 |
| content_scripts          | Content Script文件     | matches字段表示Content Script文件生效的域名                  |
| options_page             | Options文件            |                                                              |
| web_accessible_resources | 扩展需要访问的本地资源 | 只用列举的资源才能够在扩展中通过相对路径方式访问。           |

根据上面的实例文件和具体的属性介绍，相信大家对manifest文件有了一个具体的了解。下面，我们来具体介绍下我们需要使用的各个功能模块。

## 收集网页的图片（Background Pages）

需要收集各个网页的图片，我们需要一个后台常驻的脚本来满足我们的需求。因此，我们需要使用`Background Pages`。

根据前一节的manifest文件，我们指定了`background.js`文件作为我们的后台常驻脚本，下面让我们来看下这个文件的部分示例内容。

```javascript
// 注册一个右键菜单选项
chrome.runtime.onInstalled.addListener(function () {
    chrome.contextMenus.create({
        'id': 'F577D6742FF1A1AB5946A8E5281D5C5D',
        'title': '添加到表情收藏',
        'contexts': ['image']
    });
});

chrome.contextMenus.onClicked.addListener(function (info, tab) {
    var src = info['srcUrl'];
    // 获取之前存储的表情
    chrome.storage.local.get(['newFavorList'], function (items) {
        var newFavorList = items['newFavorList'] || [];
        newFavorList.push(src);
        
        // 存储所有表情
        chrome.storage.local.set({
        	'newFavorList': newFavorList
    	});
    });
});
```

通过上面的例子，我们可以实现我们的目标：当用户在任意网页上面右键一张图片时，右键菜单中都会增加一个选项——添加到表情收藏。点击这个选项，我们就能够将这张图片存储到我们的扩展应用提供的存储模块中。

其中，`runtime`和`contextMenus`是chrome提供的原生API，相关API接口可以见[此处](https://developer.chrome.com/extensions/api_index)。

具体效果如下：

![](https://user-gold-cdn.xitu.io/2018/4/8/162a5e45d3f490d8?w=682&h=454&f=png&s=489408)

## 发送保存的表情（Content Script）

当我们需要发送我们已经保存的表情时，我们就需要跟页面进行部分功能交互了。这个时候，我们需要使用到`Content Script`。

当我们使用`Content Script`时，我们的执行上下文将会是整个页面。因此，我们可以使用JavaScript来操纵DOM节点，和页面原有的JavaScript进行交互。

下面，我们通过jQuery来页面注入表情面板，同时使用PostMessage来与原有页面进行数据通信。让我们来看下`favor.js`文件的部分示例代码：

```javascript
chrome.storage.local.get(['newFavorList'], (items) => {
    let favorBox = $('#favorbox');
    favorBox.empty();
    newFavorList = items.newFavorList;


    let emotionPanel = $('<div>', {
        class: 'smiley-emotion-panel'
    });

    newFavorList.forEach((element) => {
        if (element && element.url) {
            emotionPanel.append($('<span>', {
                class: 'icon icon-smiley-emotions',
                'click': postFavor
            }).append($('<img>', {
                'width': '100%',
                'height': '100%',
                src: element.url
            })));
        }
    });

    favorBox.append(emotionPanel);
});

function postFavor(e) {
    let src = event.target.getAttribute('src');
    
    window.postMessage({
        type: 'sendCustomEmotion',
        text: src
    }, '*');
}
```

通过上面的示例代码，我们可以知道：从Storage中将表情数据取出后，立即渲染到页面中。当渲染的表情被点击时，我们就通过PostMessage将数据按照约定的格式发送即可。

在具体项目中的使用如下图所示：

![](https://user-gold-cdn.xitu.io/2018/4/8/162a5de3e633f5f9?w=443&h=194&f=png&s=128487)

这样，我们就解决了在特定的网页与页面的代码进行交互的功能。接下来让我们来看下我们的“设置”页面应该怎么开发。

## 管理已有表情（Options）

通过`Options`，我们能够给chrome扩展应用开发一个“设置”页面。当我们指定`options_page`字段后，它的值就是我们的“设置”页面。

开发一个管理已有表情的options页面，其实就是一个带有特殊API接口的网页。我们仍然能够通过chrome对象来访问chrome提供的已经申请过权限的API接口。

首先，我们将我们存储在Storage中的图片表情数据渲染出来，然后提供相关的操作函数。`options.js`部分示例代码如下：

```javascript
$scope.remove = function (obj) {
    var result = [];

    $scope.favors.forEach(function (element) {
        if (element.url !== obj.url) {
            result.push(element);
        }
    });
    $scope.favors = result;
    chrome.storage.local.set({
        'newFavorList': $scope.favors
    });
};
```

如果我们需要在“设置”页面删除后，`Content Script`页面立即响应应该怎么做呢？我们只需要在`Content Script`中增加Storage监听事件即可。具体代码示例如下：

```javascript
chrome.storage.onChanged.addListener((changes) => {
    let newFavorList = changes['newFavorList'];
    
    renderNewValue(newFavorList.newValue);
});
```

通过在`Options`和`Content Script`增加相关代码，我们就能够完成管理已有表情的功能。具体展示效果如下：

![](https://user-gold-cdn.xitu.io/2018/4/8/162a5df114fc28af?w=942&h=499&f=png&s=97524)

# 总结

我们通过一个简单的表情插件的例子来快速的介绍了chrome扩展应用的各个模块的功能和开发方法。通过这篇文章大家应该知道了chrome扩展应用各个模块的作用和开发的方法。

如果大家想对chrome扩展应用有一个更加深入的了解，那么建议自己动手开发相关的功能。这样才能够对chrome扩展应用的相关逻辑有一个更加清楚的认识。

附录中提供了部分学习相关的文档，有兴趣的同学可以自行阅读。

# 附录

- [chrome官方开发文档(英)](https://developer.chrome.com/extensions/getstarted)——建议有英文阅读能力的人阅读此文档。
- [chrome开发文档(中)](https://chajian.baidu.com/developer/extensions/getstarted.html)——阅读中文文档时，自动忽略“百度”二字即可，同时建议参考官方文档一起阅读。


