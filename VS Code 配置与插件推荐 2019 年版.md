## 概述

距离之前给大家介绍的 VS Code 插件已经过去了一年多了，在这之间有不少插件已经更新消失，也有不少的新的好用的新的插件，让我们来看下有哪些配置和插件能够提高我们的效率。

本次我们推荐的插件都是基于当前最新版本的 VS Code，即 1.31 版本。

## VS Code 配置

### `setting.json` 设置

首先，我们来看下 VS Code 推荐的设置属性。下面是我自己的配置文件：

```json
{
    "editor.fontSize": 19,
    "files.autoSave": "onFocusChange",
    "workbench.iconTheme": "vscode-icons",
    "javascript.format.insertSpaceAfterOpeningAndBeforeClosingNonemptyBraces": false,
    "typescript.format.insertSpaceAfterOpeningAndBeforeClosingNonemptyBraces": false,
    "vsicons.dontShowNewVersionMessage": true,
    "files.associations": {
        "*.css": "css"
    },
    "git.confirmSync": false,
    "workbench.activityBar.visible": true,
    "window.zoomLevel": -1,
    "typescript.check.npmIsInstalled": false,
    "git.enableSmartCommit": true,
    "explorer.confirmDragAndDrop": false,
    "git.autofetch": true,
    "explorer.confirmDelete": false,
    "editor.suggestSelection": "first",
    "editor.cursorWidth": 2,
    "javascript.implicitProjectConfig.experimentalDecorators": true,
    "editor.fontFamily": "Inziu IosevkaCC SC",
    "terminal.integrated.fontSize": 16,
    "terminal.external.osxExec": "iTerm.app",
    "workbench.editor.enablePreview": false,
    "npm.enableScriptExplorer": true,
    "editor.tabCompletion": "on",
    "sync.gist": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "editor.codeActionsOnSave": {
        "source.organizeImports": true
    }
}
```

我们来说下里面值得推荐设置的几个：

- `files.autoSave`，这个属性是表示文件是否进行自动保存，推荐设置为`onFocusChange`——文件焦点变化时自动保存。这个设置比延时保存更加方便有效。
- `editor.fontFamily`，这个属性是表示编辑区的字体，大家可以根据自己的喜好来进行设置，我推荐大家一个不错的字体 [Inziu IosevkaCC SC](http://typeof.net/Iosevka/inziu.html)，具体展示如下图：

  ![image.png](https://user-gold-cdn.xitu.io/2019/2/25/1692499e10236126?w=667&h=151&f=png&s=42061)
- `git.autofetch`，这个属性可以让 VS Code 自动去检测 Git 远端是否有发生新的更改，推荐设置为`true`。
- `editor.cursorWidth`，这个属性是用来设置光标宽度的，推荐值为`2`。在设置此属性之前，需要先设置属性`editor.cursorStyle`的值为`line`。
- `editor.suggestSelection`，这个属性是用来设置建议值的，就是我们在输入`co`时，编辑器提示我们`const`这个功能。推荐设置为`first`——即每次默认选中推荐值第一个，比如你输入`co`，推荐值为`const`和`constant`（按顺序），那么就会选择`const`。当然，我也推荐设置为`recentlyUsedByPrefix`，即上次你选择或者输入过什么，这次就默认选中什么，比如你输入`co`，推荐值为`const`和`constant`（按顺序），上次你选择了`constant`，这次就还是默认选中`cosntant`。
- `editor.tabCompletion`，这个属性是表示在出现推荐值时，按下`Tab`键是否自动填入最佳推荐值，推荐设置为`true`。
- `editor.codeActionsOnSave`中的`source.organizeImports`属性，这个属性能够在保存时，自动调整 import 语句相关顺序，能够让你的 import 语句按照字母顺序进行排列，推荐设置为`true`。

### 代码片段

在 VS Code 中，我们可以利用代码片段来避免我们复制粘贴大量重复代码块。我们通过`首选项`中的`用户代码片段`来进行代码片段的添加。具体操作路径如图所示：

![image.png](https://user-gold-cdn.xitu.io/2019/2/25/1692499fc7d346d8?w=450&h=251&f=png&s=174818)

之后，我们选择新增用户代码片段即可：

![image.png](https://user-gold-cdn.xitu.io/2019/2/25/169249a07ec9ba66?w=548&h=408&f=png&s=50313)

以我设置的代码片段为例，我是用于增加每个文件的注释头信息。因此配置如下：

```json
// typescript.json
{
	"TS description": {
		"prefix": "tsfile",
		"body": [
			"/**",
			"* @module ${TM_FILENAME_BASE}",
			"* @author: Hjava",
			"* @description: ",
			"* @since: ${CURRENT_YEAR}-${CURRENT_MONTH}-${CURRENT_DATE} ${CURRENT_HOUR}:${CURRENT_MINUTE}:${CURRENT_SECOND}",
			"*/",
			""
		],
		"description": "Insert description."
	}
}
```

文件名`typescript.json`表示了这个代码片段只在 TypeScript 语言中生效。各个属性含义如下表所示：

| 字段        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| prefix      | 前缀，即你在编辑器中输入的内容为前缀指定内容时，能够在编辑器建议中选择此片段。 |
| body        | 具体文本内容，在选择此片段后，会将此数组根据`\n`进行组合输出到编辑器中。<br />其中有部分特定的常量，可以获取输入时的部分信息，如：<br />${CURRENT\_YEAR}:当前年份，具体字段可以见[此处](https://code.visualstudio.com/docs/editor/userdefinedsnippets)<br />说明：在写此文章时，部分1.20.0版本增加的常量并不在上面的文档中，具体字段为：   `CURRENT_YEAR`：年（4位数）  `CURRENT_YEAR_SHORT`：年（2位数）  `CURRENT_MONTH`：月  `CURRENT_DATE`：日  `CURRENT_HOUR`：小时  `CURRENT_MINUTE` ：分钟 `CURRENT_SECOND`：秒 |
| description | 描述说明，在片段说明中会显示此字段的文本内容。               |

具体示例效果如下：

![](https://user-gold-cdn.xitu.io/2019/2/25/16924960f781e9f0?w=800&h=137&f=png&s=27902)

插入后效果如下：

![](https://user-gold-cdn.xitu.io/2019/2/25/169249613c667f81?w=566&h=286&f=png&s=25709)


## 插件

推荐设置属性我们介绍的差不多了，下面我们来说下推荐的插件。

### Auto Rename Tag

插件市场说明见[此处](https://marketplace.visualstudio.com/items?itemName=formulahendry.auto-rename-tag)。

这个插件的作用是能够让你的 HTML 标签能够根据你对其中某一个标签的操作自动映射到另一个对应的标签上。具体如下图所示。

![](https://user-gold-cdn.xitu.io/2019/2/25/169249613e6b2adb?w=1440&h=938&f=gif&s=158502)

### Better Comments

插件市场说明见[此处](https://marketplace.visualstudio.com/items?itemName=aaron-bond.better-comments)。

这个插件的作用是能够让你的注释看上去更加的有区分度，我们可以让注释拥有不同的颜色。具体如下图所示。

![](https://user-gold-cdn.xitu.io/2019/2/25/169249613e99b78a?w=459&h=414&f=png&s=31608)

### Document This

插件市场说明见[此处](https://marketplace.visualstudio.com/items?itemName=joelday.docthis)。

这个插件的作用是能够让你快速的对类和函数添加注释。通过按两次 `Ctrl+Alt+D` 按键可以快速增加注释。具体如下图所示。

![](https://user-gold-cdn.xitu.io/2019/2/25/169249613eabf528?w=1422&h=1078&f=gif&s=194610)

### Language PL/SQL

插件市场说明见[此处](https://marketplace.visualstudio.com/items?itemName=xyz.plsql-language)。

这个插件的作用是能够让 VS Code 支持 SQL 语言。具体如下图所示。

![](https://user-gold-cdn.xitu.io/2019/2/25/169249613ebe0b12?w=1000&h=717&f=gif&s=82899)

### Settings Sync

插件市场说明见[此处](https://marketplace.visualstudio.com/items?itemName=Shan.code-settings-sync)。

这个插件的作用是让你能够把你对于 VS Code 的配置和插件都同步到 Gist 上面，能够让你在多个电脑的 VS Code 进行同步。

### vscode-icons

插件市场说明见[此处](https://marketplace.visualstudio.com/items?itemName=robertohuertasm.vscode-icons)。

这个插件的作用是让你的文件 icon 能够具有更加丰富的样式。具体如下图所示。

![](https://user-gold-cdn.xitu.io/2019/2/25/16924990fcbc56fd?w=800&h=600&f=gif&s=2805662)

### Debugger for Chrome

插件市场说明见[此处](https://marketplace.visualstudio.com/items?itemName=msjsdiag.debugger-for-chrome)。

这个插件的作用是让你能够调用 Chrome 浏览器对自己的代码进行调试。

### ESLint

插件市场说明见[此处](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)。

这个就不用多说了，给 VS Code 增加 ESLint 的能力。

## 总结

VS Code 最近这一年更新了不少的好用的配置和插件，但是在性能上却仍然保持了一贯的优秀的水平。从目前来看，VS Code 是目前免费的编辑器中面配置最简单的开发工具。

大家如果有什么好的配置和插件，也欢迎大家留言交流。