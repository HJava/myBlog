# 背景

之前一直是只用WebStorm作为IDE来编写代码，但是由于：

1. 手中的这台Mac接了两个显示器以后，使用WebStorm会有卡顿。
2. WebStorm需要付费（虽然可以通过某方法和谐）。

所以需要寻找一个新的编辑器或者IDE来进行边写代码。

## 为什么选择VS Code

1. VS Code的性能明显由于Atom。
2. VS Code的插件系统使用方便程度远高于Sublime。
3. VS Code相对于WebStorm来说是免费的。

# VS Code配置

**注：当前VS Code相关的配置基于v1.20.1版本。**

## 用户设置

在`首选项->设置`中，能够对VS Code相关的属性进行设置，目前有调整字段如下：

- `"editor.fontSize": 16`，该设置用来调整编辑器中的字体大小，目前设置大小为16。
- `"files.autoSave": "onFocusChange"`，该设置用来调整编辑器的自动保存策略，当前字段含义为当该文件失焦后保存，即切换到其他应用或者文件的时候自动进行一次保存。
- `"editor.cursorWidth": 2`，该设置是用来控制光标的粗细，目前设置大小为2。
- `"editor.suggestSelection": "recentlyUsedByPrefix"`，该设置是用来控制自动补全的建议，目前设置为根据之前补全过建议的前缀来进行建议，大概的意思就是你上次通过`co`选择了`const`，这次你再输入`co`的时候，也会建议你选择`const`。

## 代码片段

VS Code可以通过名为`代码片段`的功能像编辑器中插入一段指定的文本，具体操作步骤为`首选项->用户代码片段->新建全局代码片段`。

我们可以增加一些常用的文件声明注释、通用模板等代码片段，从而避免频繁的复制粘贴和重复劳动。

我举一个简单的`文件声明注释`的例子来说明下这个功能：

```json
{
	// Place your snippets for javascript here. Each snippet is defined under a snippet name and has a prefix, body and 
	// description. The prefix is what is used to trigger the snippet and the body will be expanded and inserted. Possible variables are:
	// $1, $2 for tab stops, $0 for the final cursor position, and ${1:label}, ${2:another} for placeholders. Placeholders with the 
	// same ids are connected.
	// Example:
	// "Print to console": {
	// 	"prefix": "log",
	// 	"body": [
	// 		"console.log('$1');",
	// 		"$2"
	// 	],
	// 	"description": "Log output to console"
	// }
	"JS & TS description": {
		"prefix": "jsfile",
		"body": [
			"/**",
			"* @module ${TM_FILENAME_BASE}",
			"* @author: Hjava",
			"* @since: ${CURRENT_YEAR}-${CURRENT_MONTH}-${CURRENT_DATE} ${CURRENT_HOUR}:${CURRENT_MINUTE}:${CURRENT_SECOND}",
			"*/",
			"",
			"'use strict';",
			""
		],
		"description": "Insert description."
	}
}
```

其中，`JS & TS description`表示这个片段的名称，其他具体字段含义如下表所示：

| 字段        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| prefix      | 前缀，即你在编辑器中输入的内容为前缀指定内容时，能够在编辑器建议中选择此片段。 |
| body        | 具体文本内容，在选择此片段后，会将此数组根据`\n`进行组合输出到编辑器中。<br />其中有部分特定的常量，可以获取输入时的部分信息，如：<br />${CURRENT_YEAR}:当前年份，具体字段可以见[此处](https://code.visualstudio.com/docs/editor/userdefinedsnippets)<br />说明：在写此文章时，部分1.20.0版本增加的常量并不在上面的文档中，具体字段为：   `CURRENT_YEAR`：年（4位数）  `CURRENT_YEAR_SHORT`：年（2位数）  `CURRENT_MONTH`：月  `CURRENT_DATE`：日  `CURRENT_HOUR`：小时  `CURRENT_MINUTE` ：分钟 `CURRENT_SECOND`：秒 |
| description | 描述说明，在片段说明中会显示此字段的文本内容。               |

具体示例效果如下：

![](https://segmentfault.com/img/bV3048)

插入后效果如下：

![](https://segmentfault.com/img/bV305c)

## 插件

在左侧插件面板中，可以进行插件的搜索、安装与卸载。推荐插件如下：

- `Auto Close Tag`，能够在你编写HTML中自动帮你加上闭合的标签。
- `Auto Rename Tag`，能够在你修改一个标签时自动调整与之成对的另一个标签。
- `js-beautify for VS Code`，能够格式化你的JavaScript文件。当然，它还提供了格式化JSON的能力。
- `Beautify css/sass/scss/less`，它能够让你对CSS相关文件进行格式化。
- `Better Comments`，能够让你的注释看上去更加友好。
- `Document This`，能够自动的给函数和方法添加注释。
- `ESLint`，这个不用多说，给VS Code提供了ESLint相关功能。
- `PostCSS Syntax Highlighting`，能够让VS Code支持PostCSS语法。
- `vscode-icons`，能够让你的文件树增加图标标识。

# 总结

VS Code总体上来说是一个使用比较方便的编辑器，能够通过一些特定的插件提高你的工作效率。相较于其他的IDE或者编辑器来看，他有着自己独特的优势。