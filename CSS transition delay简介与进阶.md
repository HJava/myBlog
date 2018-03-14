# 背景
在日常的项目开发中，我们会很经常的遇见如下的需求：

- 在浏览器页面中，当鼠标移动到某个部分后，另一个部分在延迟若干时间后出现
- 在鼠标移除该区域后，另一部分也在延迟若干时间后消失

我相信这是一个很常见的一个需求，有很多种方式能够实现，但是，其实现方式的原理各不相同，也有利有弊。
# 实现方案
## CSS
在CSS中，有一个伪类`hover`也能够监听鼠标移动到某个元素上面，因此我们也可以利用CSS来实现我们刚刚的功能。

我们需要使用的是CSS3中的新特性：`transition`。

`transition`是一个简写属性，可设置`transition-property`, `transition-duration`, `transition-timing-function`,  `transition-delay`。 `transition`用来定义元素两种状态之间的过渡。不同状态可以用 pseudo-classes 定义，比如 `:hover` 、`:active`或使用JavaScript设置。

该属性第一个值为需要变化的属性值，第二个值为其持续时间，第三个值为变化方式，第四个值为其延时。该属性指定的值只要指定的属性有任何变化，都会触发该属性。即在从该样式到其他样式，以及其他样式回到该样式时都会产生效果。例如：

	transition:opacity 1s linear 1s;
	
具体介绍请看[MDN官方介绍](https://developer.mozilla.org/zh-CN/docs/Web/CSS/transition)。

现在，让我们用`transition`属性来实现上面的功能。简单点如下所示：

	//HTML文件
	<!DOCTYPE html>
	<html lang="en">
	<head>
	    <meta charset="UTF-8">
	    <title>Title</title>
	    <link rel="stylesheet" href="css/index.css">
	</head>
	<body>
	<div id="container">
	    <div id="div1"></div>
	    <div id="div2"></div>
	</div>
	</body>
	</html>

	//Sass文件
	#container {
	  display: inline-block;
	  #div1 {
	    width: 100px;
	    height: 100px;
	    background-color: #000;
	  }
	
	  #div2 {
	    visibility: hidden;
	    opacity: 0;
	    width: 100px;
	    height: 100px;
	    background-color: #F00;
	    transition-delay: 0.5s;
	  }
	  &:hover {
	    #div2 {
	      visibility: visible;
	      opacity: 1;
	    }
	  }
	}
	
通过在CSS中添加`transition-duration`，我们可以实现延时显示的目的。并且，我们不需要自己去清除定时器，而由浏览器来判别。

在此，我们为什么不用`display`属性呢，因为在`display`改变时，`transition`并不会生效。

那我们为什么需要在使用了`opacity`属性的时候同时使用`visibility`属性呢。因为`opacity`属性只是让元素变得透明，而不会让元素消失。如果不加速`visibility`属性的话，那元素变透明后仍然可以点击，那么会出现一些奇怪的影响。

到目前为止，我们利用CSS完全模拟了第一部分我们使用JavaScript实现的功能，而且看上去更简洁。那么，下面我们来讲一些更加复杂的功能有助于大家理解`transition`。

## transition进阶
现在我们需要在鼠标移动上去后，出现一个渐变的效果，让另一框慢慢出现，同时在鼠标移出的时候也有渐变消失的效果，那么我们就需要来使用一下`transition`的另外几个属性。

在上面的代码中稍加改动，就能够得到我们需要的[效果](http://jsbin.com/vukovatiye/edit?html,css,output)。

	transition: opacity 0.5s linear;
	
这样的话，在鼠标移入的时候，会有一个渐变的效果。但是，问题也出现了，在鼠标移出的时候，`div2`立马就消失了。让我们来分析一下鼠标移入和移出时的效果。

当鼠标移入时：

1. 鼠标移入`div1`元素
2. 触发了`hover`事件，div2的`visibility`属性变为`visible`
3. `transition`属性让`opacity`属性从0变为1

而当鼠标移出时：

1. 鼠标移出`div1`元素
2. `hover`事件停止触发，div2的`visibility`属性变为`hidden`
3. `transition`属性让`opacity`属性从1变为0


根据上面的情况我们知道，当`hover`事件结束后，`visibility`属性立马就变成了`hidden`了，因此后面的动画效果就无法看到了。

那么如果我们为`visibility`属性也加上延时呢，能不能够达到我们的目标，让我们来看一下[效果](http://jsbin.com/lugugavusu/edit?html,css,output)。

	    transition: visibility 0s linear 0.5s, opacity 0.5s linear;
		
代码改动为如上情况后，我们会发现，当鼠标移出的时候，能够看到动画效果。但是当鼠标移入时，动画效果消失了，现在再让我们来分析下为什么会出现这个情况。

当时鼠标移入时：

1. 鼠标移入`div1`元素
2. 触发`hover`事件
3. `transition`属性让`opacity`属性从0变为1
4. `visibility`属性变为`visible`

当鼠标移出时：

1. 鼠标移出`div1`元素
2. `hover`事件停止触发
3. `transition`属性让`opacity`属性从1变为0
4. `visibility`属性变为`hidden`

从上面的分析我们可以知道，因为`visibility`属性为不连续变化属性，因此在`transition`中只有`transition-delay`对该属性产生了效果。所以`visibility`属性延时了0.5s执行，导致了在鼠标移入时看不到效果。

那么，我们有没有办法同时在鼠标移入和移出的时候同时看到动画效果呢。需要达到这个目的，其实换一个思路立马就能够解决。我们不只需要在`hover`事件中重置这个延时，将其重新指定为0，马上就能够达到我们想要的[效果](http://jsbin.com/lugugavusu/edit?html,css,output)。具体示例代码如下：
	
	#container {
	  display: inline-block;
	  #div1 {
	    width: 100px;
	    height: 100px;
	    background-color: #000;
	  }
	
	  #div2 {
	    visibility: hidden;
	    opacity: 0;
	    width: 100px;
	    height: 100px;
	    background-color: #F00;
	    transition: visibility 0s linear 0.5s, opacity 0.5s linear;
	  }
	  &:hover {
	    #div2 {
	      visibility: visible;
	      opacity: 1;
	      transition-delay: 0s;
	    }
	  }
	}
	
那么现在让我们来分析下为什么这么写能够达到我们需要的效果。

当鼠标移入时：

1. 鼠标移入`div1`
2. `hover`事件触发，重新指定`transition`属性的`transition-delay`为0s，`visibility`属性由`hidden`立马变成`visible`
3. `transition`属性让`opacity`属性由0变为1

当鼠标移出时：

1. 鼠标移出`div1`
2. `hover`事件停止触发，`transition`延时恢复到0.5s，因此`visibility`属性不会马上触发
3. `transition`属性让`opacity`属性由1变为0
4. `visibility`属性由`visible`变为`hidden`

我们通过一个很简单巧妙的方法，通过改变`transition-delay`，从而让`visibility`属性立即执行，达到了我们需要的效果。

## JavaScript
附上利用JS来实现该方案的代码用于参考。这个其实应该是大部分人会想到的方法，利用`mouseover`或者`click`事件来进行事件的监听，利用`setTimeout`来进行延时处理，例如[这样](http://jsbin.com/huqijufano/edit?html,css,js,output)：

	<!DOCTYPE html>
	<html lang="en">
	<head>
	    <meta charset="UTF-8">
	    <title>Title</title>
	    <style>
	        #div1 {
	            width: 100px;
	            height: 100px;
	            background-color: #000;
	        }
	
	        #div2 {
	            display: none;
	            width: 100px;
	            height: 100px;
	            background-color: #F00;
	        }
	    </style>
	</head>
	<body>
	<div id="div1"></div>
	<div id="div2"></div>
	<script>
	    document.addEventListener('DOMContentLoaded', function() {
	        var div1 = document.getElementById('div1');
	        var div2 = document.getElementById('div2');
	
	        div1.addEventListener('mouseover', function() {
	            setTimeout(function() {
	                div2.style.display = 'block';
	            }, 500);
	        });
	
	        div1.addEventListener('mouseleave', function() {
	            setTimeout(function() {
	                div2.style.display = 'none';
	            }, 500);
	        });
	    });
	</script>
	</body>
	</html>
	
但是上面这个代码有一个比较严重的问题，就是当鼠标两次移动上去的间隔小于500ms时，上一次的`setTimeout`的代码还是会触发，因此会看到一次闪动的效果。

因此，我们需要在检测到两次间隔小于500ms时，清除掉上次的`setTimeout`的代码。所以，改动代码[如下](http://jsbin.com/pafejavedo/edit?html,css,js,output)(注：如有表现与描述不一致请看文章末尾说明):

	div1.addEventListener('mouseover', function() {
	    clearTimeout(handle);
	    setTimeout(function() {
	        div2.style.display = 'block';
	    }, 500);
	});
	
	var handle = 0;
	div1.addEventListener('mouseleave', function() {
	    handle = setTimeout(function() {
	        div2.style.display = 'none';
	    }, 500);
	});
	
调整为上述代码之后，基本可以满足我们的需求。后续如果需要添加动画之类的操作，也只需要继续像代码添加相关逻辑即可，在此就不再演示。

# 总结
在需求开发的过程中，遇到了这个问题。最开始用JavaScript实现，开发起来比较复杂，容易与业务逻辑代码混在一起不好维护。通过CSS来实现这个功能，既简单高效，又能够避免增加JavaScript复杂度，是一个比较优的解决方案。
# 参考
- [CSS3 transitions using visibility and delay(英文)](http://www.greywyvern.com/?post=337)
- [transition过渡](https://developer.mozilla.org/zh-CN/docs/Web/CSS/transition)
- [transition示例](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Transitions/Using_CSS_transitions)

# 说明
[jsbin](http://jsbin.com/?html,css,output)是一个在线的编辑器，能够在代码编写完成后马上看到效果，但是该文中有部分代码在jsbin中出现表现与本地不一致的情况(例如CSS进阶中最后一个代码)，大家可以将代码放到本地验证。由于代码效果时好时坏，猜测可能与jsbin的容器相关。