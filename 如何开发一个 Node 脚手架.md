# 原因

在工作中，需要开发一个脚手架，用于给相关用户提供相关的开发便利性。

# 适合人群

对前端、Node 操作有一定的了解，同时想了解脚手架开发过程或者需要自己实现一个脚手架的开发者。

# 目标

1. 开发一个简单的脚手架，能够提供给用户进行安装。
2. 能够输出相关提示。
3. 对用户文件进行读写操作。
4. 在脚手架中使用 Shell 脚本。

# 步骤

## 开发脚手架

脚手架的开发最开始过程与普通的前端项目相同，需要一个入口文件 `command.js` 和配置文件 `package.json`。

与其他配置文件不同的是，你需要在 `command.js` 文件第一行增加如下声明：

```
#! /usr/bin/env node
```

同时需要在 `package.json` 文件中加上一下一项：

```javascript
{
  ...,
  "bin": {
   	"cm-cli": "command.js"
   }
}
```

在配置文件中增加了此项后，只需要在配置文件根目录下执行 `npm link` 命令，即可使用 `cm-cli --help` 命令来查看加载的 cm-cli 脚手架（需要保证 command.js 能够处理响应，详情见下一节，放在此处是为了文章方便阅读）。

如果你发布了你的脚手架，那么在其他用户使用命令 `npm install -g cm-cli` 之后，便可以在全局下使用你的脚手架了。

## 对用户进行提示

在对注释和命令进行提示中，我们需要使用到 `commander` 包，使用 `npm install commander` 即可进行安装。（如果NPM版本低于5，则需要添加 `--save` 参数保证更新 `package.json` 配置文件）。

`commander` 是一个提供用户命令行输入和参数解析的强大功能。有需要的可以阅读相关的库文档。在这里我介绍两个用的最多的方法。

### option

能够初始化自定义的参数对象，设置关键字和描述，同时还可以设置读取用户输入的参数。具体用法如下：

```javascript
const commander = require('commander');

commander.version('1.0.0')
    .option('-a, --aaa', 'aaaaa')
    .option('-b, --bbb', 'bbbbb')
    .option('-c, --ccc [name]', 'ccccc')
    .parse(process.argv);


if (commander.aaa) {
    console.log('aaa');
}

if (commander.bbb) {
    console.log('bbb');
}

if (commander.ccc) {
    console.log('ccc', commander.ccc);
}
```

此时如果已经配置完成并且执行过 `npm link` 命令后，可以看到如下结果：

![clipboard.png][image-1]

### command

该方法能够在命令行增加一个命令。用户在执行此命令后，能够执行回调中的逻辑。具体用法如下：

```javascript
commander
    .command('init <extensionId>')
    .description('init extension project')
    .action((extensionId) => {
        console.log(`init Extension Project "${extensionId}"`);
		// todo something you need
    });
```

具体展示效果如下：

![][image-2]

## 对用户文件进行读写操作

通过上面的步骤，我们已经能够完成一个简单的脚手架了。下面，我们需要读取用户配置，同时为用户生成一些模板文件。

### 读取文件

现在，我们需要读取用户的 `cm-cli.json` 配置文件来进行一些配置。

我们可以使用 Node.js 的 `fs` 文件模块来对文件进度读操作，由于此处没有太多难点，因此略去。

### 写入文件模板

我们提前将模板文件存储在 CDN 上，再根据本地读取到的相关脚手架配置文件来进行模板的下载。

**注：脚手架中读取的路径为使用者使用时当前路径，因此没有办法将模板文件存储在脚手架中进行读取。**

我们可以使用诸如 `request` 这种库来帮助我们进行文件下载，简化操作步骤。执行 `npm install request` 即可进行安装。

**注：在文件写入时建议先判断文件是否存在，再进行覆盖。**

## 使用 Shell 脚本

与 Node.js 提供的 API 函数来看，有些人更加倾向于使用 Shell 脚本来进行文件操作。幸运的是，我们也可以在我们的脚手架中引入 `node-cmd` 来启用对 Shell 脚本的支持。执行 `npm install node-cmd` 即可进行安装。

具体示例如下：

```javascript
commander
    .command('init <extensionId>')
    .description('init extension project')
    .action((extensionId) => {
        id = extensionId;
        console.log(`init Extension Project "${extensionId}"`);

        cmd.get(
            `
            mkdir -p static/${extensionId}

            mkdir tmp
            mkdir tmp/source-file
            mkdir tmp/build-file
            curl -o tmp/source-file/index.js https://xxxxxxxx.com?filename=index.js
            touch tmp/source-file/index.css

            curl -o tmp/build-file/server.js https://xxxxxxxx.com?filename=server.js
            curl -o tmp/build-file/router.js https://xxxxxxxx.com?filename=router.js
            curl -o tmp/build-file/package.json https://xxxxxxxx.com?filename=package.json
            
            cp  tmp/source-file/* static/${extensionId}
            cp tmp/build-file/* ./
            rm -fr tmp
            npm install
            `,
            (err, data) => {
                console.log(data)
                if (!err) {
                    console.log('init success');
                    return;
                }

                console.error('init error');
            });
    });
```

我们可以快速的使用 Shell 脚本来进行文件夹的创建和文件模板的下载。

# 总结

脚手架想要在终端能够快速执行，可以在 `package.json` 配置文件中增加相关字段。

脚手架需要能够读取相关终端输入，可以使用 `commander` 库来快速开发。

脚手架需要能够执行 Shell 脚本，可以使用 `node-cmd` 库来快速实现需求。

[image-1]:	https://segmentfault.com/img/bVZB2r
[image-2]:	https://segmentfault.com/img/bVZB24