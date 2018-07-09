# 概述

本文通过对[virtual-dom](https://github.com/Matt-Esch/virtual-dom)的源码进行阅读和分析，针对Virtual DOM的结构和相关的Diff算法进行讲解，让读者能够对整个数据结构以及相关的Diff算法有一定的了解。

Virtual DOM中Diff算法得到的结果如何映射到真实DOM中，我们将在下一篇博客揭晓。

本文的主要内容为：

- Virtual DOM的结构
- Virtual DOM的Diff算法

注：这个Virtual DOM的实现并不是React Virtual DOM的源码，而是基于[virtual-dom]((https://github.com/Matt-Esch/virtual-dom))这个库。两者在原理上类似，并且这个库更加简单容易理解。相较于这个库，React对Virtual DOM做了进一步的优化和调整，我会在后续的博客中进行分析。

# Virtual DOM的结构

## VirtualNode

作为Virtual DOM的元数据结构，VirtualNode位于`vnode/vnode.js`文件中。我们截取一部分声明代码来看下内部结构：

```javascript
function VirtualNode(tagName, properties, children, key, namespace) {
    this.tagName = tagName
    this.properties = properties || noProperties //props对象，Object类型
    this.children = children || noChildren //子节点，Array类型
    this.key = key != null ? String(key) : undefined
    this.namespace = (typeof namespace === "string") ? namespace : null
    
    ...

    this.count = count + descendants
    this.hasWidgets = hasWidgets
    this.hasThunks = hasThunks
    this.hooks = hooks
    this.descendantHooks = descendantHooks
}

VirtualNode.prototype.version = version //VirtualNode版本号，isVnode()检测标志
VirtualNode.prototype.type = "VirtualNode" // VirtualNode类型，isVnode()检测标志
```

上面就是一个VirtualNode的完整结构，包含了特定的标签名、属性、子节点等。

## VText

VText是一个纯文本的节点，对应的是HTML中的纯文本。因此，这个属性也只有`text`这一个字段。

```javascript
function VirtualText(text) {
    this.text = String(text)
}

VirtualText.prototype.version = version
VirtualText.prototype.type = "VirtualText"

```

## VPatch

VPatch是表示需要对Virtual DOM执行的操作记录的数据结构。它位于`vnode/vpatch.js`文件中。我们来看下里面的具体代码：

```javascript
// 定义了操作的常量，如Props变化，增加节点等
VirtualPatch.NONE = 0
VirtualPatch.VTEXT = 1
VirtualPatch.VNODE = 2
VirtualPatch.WIDGET = 3
VirtualPatch.PROPS = 4
VirtualPatch.ORDER = 5
VirtualPatch.INSERT = 6
VirtualPatch.REMOVE = 7
VirtualPatch.THUNK = 8

module.exports = VirtualPatch

function VirtualPatch(type, vNode, patch) {
    this.type = Number(type) //操作类型
    this.vNode = vNode //需要操作的节点
    this.patch = patch //需要操作的内容
}

VirtualPatch.prototype.version = version
VirtualPatch.prototype.type = "VirtualPatch"
```

其中常量定义了对VNode节点的操作。例如：VTEXT就是增加一个VText节点，PROPS就是当前节点有Props属性改变。

# Virtual DOM的Diff算法

了解了虚拟DOM中的三个结构，那我们下面来看下Virtual DOM的Diff算法。

这个Diff算法是Virtual DOM中最核心的一个算法。通过输入初始状态A（VNode）和最终状态B（VNode），这个算法可以得到从A到B的变化步骤（VPatch），根据得到的这一连串步骤，我们就可以知道哪些节点需要新增，哪些节点需要删除，哪些节点的属性有了变化。在这个Diff算法中，又分成了三部分：

- VNode的Diff算法
- Props的Diff算法
- Vnode children的Diff算法

下面，我们就来一个一个介绍这些Diff算法。

## VNode的Diff算法

该算法是针对于单个VNode的比较算法。它是用于两个树中单个节点比较的场景。具体算法如下，如果不想直接阅读源码的同学也可以翻到下面，会有相关代码流程说明供大家参考：

```javascript
function walk(a, b, patch, index) {
    if (a === b) {
        return
    }

    var apply = patch[index]
    var applyClear = false

    if (isThunk(a) || isThunk(b)) {
        thunks(a, b, patch, index)
    } else if (b == null) {

        // If a is a widget we will add a remove patch for it
        // Otherwise any child widgets/hooks must be destroyed.
        // This prevents adding two remove patches for a widget.
        if (!isWidget(a)) {
            clearState(a, patch, index)
            apply = patch[index]
        }

        apply = appendPatch(apply, new VPatch(VPatch.REMOVE, a, b))
    } else if (isVNode(b)) {
        if (isVNode(a)) {
            if (a.tagName === b.tagName &&
                a.namespace === b.namespace &&
                a.key === b.key) {
                var propsPatch = diffProps(a.properties, b.properties)
                if (propsPatch) {
                    apply = appendPatch(apply,
                        new VPatch(VPatch.PROPS, a, propsPatch))
                }
                apply = diffChildren(a, b, patch, apply, index)
            } else {
                apply = appendPatch(apply, new VPatch(VPatch.VNODE, a, b))
                applyClear = true
            }
        } else {
            apply = appendPatch(apply, new VPatch(VPatch.VNODE, a, b))
            applyClear = true
        }
    } else if (isVText(b)) {
        if (!isVText(a)) {
            apply = appendPatch(apply, new VPatch(VPatch.VTEXT, a, b))
            applyClear = true
        } else if (a.text !== b.text) {
            apply = appendPatch(apply, new VPatch(VPatch.VTEXT, a, b))
        }
    } else if (isWidget(b)) {
        if (!isWidget(a)) {
            applyClear = true
        }

        apply = appendPatch(apply, new VPatch(VPatch.WIDGET, a, b))
    }

    if (apply) {
        patch[index] = apply
    }

    if (applyClear) {
        clearState(a, patch, index)
    }
}
```

代码具体逻辑如下：

1. 如果`a`和`b`这两个VNode全等，则认为没有修改，直接返回。
2. 如果其中有一个是thunk，则使用thunk的比较方法`thunks`。
3. 如果`a`是widget且`b`为空，那么通过递归将`a`和它的子节点的`remove`操作添加到patch中。
4. 如果`b`是VNode的话，
    1. 如果`a`也是VNode，那么比较`tagName`、`namespace`、`key`，如果相同则比较两个VNode的Props（用下面提到的diffProps算法），同时比较两个VNode的children（用下面提到的diffChildren算法）；如果不同则直接将`b`节点的`insert`操作添加到patch中，同时将标记位置为true。
    2. 如果`a`不是VNode，那么直接将`b`节点的`insert`操作添加到patch中，同时将标记位置为true。
5. 如果`b`是VText的话，看`a`的类型是否为VText，如果不是，则将`VText`操作添加到patch中，并且将标志位设置为true；如果是且文本内容不同，则将`VText`操作添加到patch中。
6. 如果`b`是Widget的话，看`a`的类型是否为widget，如果是，将标志位设置为true。不论`a`类型为什么，都将`Widget`操作添加到patch中。
7. 检查标志位，如果标识为为true，那么通过递归将`a`和它的子节点的`remove`操作添加到patch中。

这就是单个VNode节点的diff算法全过程。这个算法是整个diff算法的入口，两棵树的比较就是从这个算法开始的。

## Prpps的Diff算法

看完了单个VNode节点的diff算法，我们来看下上面提到的`diffProps`算法。

该算法是针对于两个比较的VNode节点的Props比较算法。它是用于两个场景中key值和标签名都相同的情况。具体算法如下，如果不想直接阅读源码的同学也可以翻到下面，会有相关代码流程说明供大家参考：

```javascript
function diffProps(a, b) {
    var diff

    for (var aKey in a) {
        if (!(aKey in b)) {
            diff = diff || {}
            diff[aKey] = undefined
        }

        var aValue = a[aKey]
        var bValue = b[aKey]

        if (aValue === bValue) {
            continue
        } else if (isObject(aValue) && isObject(bValue)) {
            if (getPrototype(bValue) !== getPrototype(aValue)) {
                diff = diff || {}
                diff[aKey] = bValue
            } else if (isHook(bValue)) {
                 diff = diff || {}
                 diff[aKey] = bValue
            } else {
                var objectDiff = diffProps(aValue, bValue)
                if (objectDiff) {
                    diff = diff || {}
                    diff[aKey] = objectDiff
                }
            }
        } else {
            diff = diff || {}
            diff[aKey] = bValue
        }
    }

    for (var bKey in b) {
        if (!(bKey in a)) {
            diff = diff || {}
            diff[bKey] = b[bKey]
        }
    }

    return diff
}
```

代码具体逻辑如下：

1. 遍历`a`对象。
    1. 当key值不存在于`b`，则将此值存储下来，value赋值为`undefined`。
    2. 当此key对应的两个属性都相同时，继续终止此次循环，进行下次循环。
    3. 当key值对应的value不同且key值对应的两个value都是对象时，判断Prototype值，如果不同则记录key对应的`b`对象的值；如果`b`对应的value是`hook`的话，记录b的值。
    4. 上面条件判断都不同且都是对象时，则继续比较key值对应的两个对象（递归）。
    5. 当有一个不是对象时，直接将`b`对应的value进行记录。
2. 遍历`b`对象，将所有`a`对象中不存在的key值对应的对象都记录下来。

整个算法的大致流程如下，因为比较简单，就不画相关流程图了。如果逻辑有些绕的话，可以配合代码食用，效果更佳。

## Vnode children的Diff算法

下面让我们来看下最后一个算法，就是关于两个VNode节点的children属性的`diffChildren`算法。这个个diff算法分为两个部分，第一部分是将变化后的结果`b`的children进行顺序调整的算法，保证能够快速的和`a`的children进行比较；第二部分就是将`a`的children与重新排序调整后的`b`的children进行比较，得到相关的patch。下面，让我们一个一个算法来看。

## reorder算法

该算法的作用是将`b`节点的children数组进行调整重新排序，让`a`和`b`两个children之间的diff算法更加节约时间。具体代码如下：

```javascript
function reorder(aChildren, bChildren) {
    // O(M) time, O(M) memory
    var bChildIndex = keyIndex(bChildren)
    var bKeys = bChildIndex.keys  // have "key" prop，object
    var bFree = bChildIndex.free  //don't have "key" prop，array

    // all children of b don't have "key"
    if (bFree.length === bChildren.length) {
        return {
            children: bChildren,
            moves: null
        }
    }

    // O(N) time, O(N) memory
    var aChildIndex = keyIndex(aChildren)
    var aKeys = aChildIndex.keys
    var aFree = aChildIndex.free

    // all children of a don't have "key"
    if (aFree.length === aChildren.length) {
        return {
            children: bChildren,
            moves: null
        }
    }

    // O(MAX(N, M)) memory
    var newChildren = []

    var freeIndex = 0
    var freeCount = bFree.length
    var deletedItems = 0

    // Iterate through a and match a node in b
    // O(N) time,
    for (var i = 0 ; i < aChildren.length; i++) {
        var aItem = aChildren[i]
        var itemIndex

        if (aItem.key) {
            if (bKeys.hasOwnProperty(aItem.key)) {
                // Match up the old keys
                itemIndex = bKeys[aItem.key]
                newChildren.push(bChildren[itemIndex])

            } else {
                // Remove old keyed items
                itemIndex = i - deletedItems++
                newChildren.push(null)
            }
        } else {
            // Match the item in a with the next free item in b
            if (freeIndex < freeCount) {
                itemIndex = bFree[freeIndex++]
                newChildren.push(bChildren[itemIndex])
            } else {
                // There are no free items in b to match with
                // the free items in a, so the extra free nodes
                // are deleted.
                itemIndex = i - deletedItems++
                newChildren.push(null)
            }
        }
    }

    var lastFreeIndex = freeIndex >= bFree.length ?
        bChildren.length :
        bFree[freeIndex]

    // Iterate through b and append any new keys
    // O(M) time
    for (var j = 0; j < bChildren.length; j++) {
        var newItem = bChildren[j]

        if (newItem.key) {
            if (!aKeys.hasOwnProperty(newItem.key)) {
                // Add any new keyed items
                // We are adding new items to the end and then sorting them
                // in place. In future we should insert new items in place.
                newChildren.push(newItem)
            }
        } else if (j >= lastFreeIndex) {
            // Add any leftover non-keyed items
            newChildren.push(newItem)
        }
    }

    var simulate = newChildren.slice()
    var simulateIndex = 0
    var removes = []
    var inserts = []
    var simulateItem

    for (var k = 0; k < bChildren.length;) {
        var wantedItem = bChildren[k]
        simulateItem = simulate[simulateIndex]

        // remove items
        while (simulateItem === null && simulate.length) {
            removes.push(remove(simulate, simulateIndex, null))
            simulateItem = simulate[simulateIndex]
        }

        if (!simulateItem || simulateItem.key !== wantedItem.key) {
            // if we need a key in this position...
            if (wantedItem.key) {
                if (simulateItem && simulateItem.key) {
                    // if an insert doesn't put this key in place, it needs to move
                    if (bKeys[simulateItem.key] !== k + 1) {
                        removes.push(remove(simulate, simulateIndex, simulateItem.key))
                        simulateItem = simulate[simulateIndex]
                        // if the remove didn't put the wanted item in place, we need to insert it
                        if (!simulateItem || simulateItem.key !== wantedItem.key) {
                            inserts.push({key: wantedItem.key, to: k})
                        }
                        // items are matching, so skip ahead
                        else {
                            simulateIndex++
                        }
                    }
                    else {
                        inserts.push({key: wantedItem.key, to: k})
                    }
                }
                else {
                    inserts.push({key: wantedItem.key, to: k})
                }
                k++
            }
            // a key in simulate has no matching wanted key, remove it
            else if (simulateItem && simulateItem.key) {
                removes.push(remove(simulate, simulateIndex, simulateItem.key))
            }
        }
        else {
            simulateIndex++
            k++
        }
    }

    // remove all the remaining nodes from simulate
    while(simulateIndex < simulate.length) {
        simulateItem = simulate[simulateIndex]
        removes.push(remove(simulate, simulateIndex, simulateItem && simulateItem.key))
    }

    // If the only moves we have are deletes then we can just
    // let the delete patch remove these items.
    if (removes.length === deletedItems && !inserts.length) {
        return {
            children: newChildren,
            moves: null
        }
    }

    return {
        children: newChildren,
        moves: {
            removes: removes,
            inserts: inserts
        }
    }
}
```

下面，我们来简单介绍下这个排序算法：

1. 检查`a`和`b`中的children是否拥有key字段，如果没有，直接返回`b`的children数组。
2. 如果存在，初始化一个数组newChildren，遍历`a`的children元素。
    1. 如果aChildren存在key值，则去bChildren中找对应key值，如果bChildren存在则放入新数组中，不存在则放入一个null值。
    2. 如果aChildren不存在key值，则从bChildren中不存在key值的第一个元素开始取，放入新数组中。
3. 遍历bChildren，将所有achildren中没有的key值对应的value或者没有key，并且没有放入新数组的子节点放入新数组中。
4. 将bChildren和新数组逐个比较，得到从新数组转换到bChildren数组的`move`操作patch（即`remove`+`insert`）。
5. 返回新数组和`move`操作列表。

通过上面这个排序算法，我们可以得到一个新的`b`的children数组。在使用这个数组来进行比较厚，我们可以将两个children数组之间比较的时间复杂度从o(n^2)转换成o(n)。具体的方法和效果我们可以看下面的DiffChildren算法。

## DiffChildren算法

```javascript
function diffChildren(a, b, patch, apply, index) {
    var aChildren = a.children
    var orderedSet = reorder(aChildren, b.children)
    var bChildren = orderedSet.children

    var aLen = aChildren.length
    var bLen = bChildren.length
    var len = aLen > bLen ? aLen : bLen

    for (var i = 0; i < len; i++) {
        var leftNode = aChildren[i]
        var rightNode = bChildren[i]
        index += 1

        if (!leftNode) {
            if (rightNode) {
                // Excess nodes in b need to be added
                apply = appendPatch(apply,
                    new VPatch(VPatch.INSERT, null, rightNode))
            }
        } else {
            walk(leftNode, rightNode, patch, index)
        }

        if (isVNode(leftNode) && leftNode.count) {
            index += leftNode.count
        }
    }

    if (orderedSet.moves) {
        // Reorder nodes last
        apply = appendPatch(apply, new VPatch(
            VPatch.ORDER,
            a,
            orderedSet.moves
        ))
    }

    return apply
}
```

通过上面的重新排序算法整理了以后，两个children比较就只需在相同下标的情况下比较了，即aChildren的第N个元素和bChildren的第N个元素进行比较。然后较长的那个元素做`insert`操作（bChildren）或者`remove`操作（aChildren）即可。最后，我们将move操作再增加到patch中，就能够抵消我们在reorder时对整个数组的操作。这样只需要一次便利就得到了最终的patch值。

# 总结

整个Virtual DOM的diff算法设计的非常精巧，通过三个不同的分部算法来实现了VNode、Props和Children的diff算法，将整个Virtual DOM的的diff操作分成了三类。同时三个算法又互相递归调用，对两个Virtual DOM数做了一次（伪）深度优先的递归比较。

下面一片博客，我会介绍如何将得到的VPatch操作应用到真实的DOM中，从而导致HTML树的变化。