# 概述

本文通过对`IndexedDB`的使用方法和使用场景进行相关介绍，对常见的问题进行解答。

同时，因为MDN中的相关文档缺乏相关逻辑性，所以不容易理解。本文将通过项目中常见的数据存储和操作需求来进行内容组织。

读者能够通过本文学会在项目中正确的使用`IndexedDB`，给应用带来的本地存储能力，并且避免一些常见的问题。

# 原因：开发者需要在本地进行永久存储

当我们进行一些较大的SPA页面开发时，我们会需要进行一些数据的本地存储。

当数据量不大时，我们可以通过SessionStorage或者LocalStorage来进行存储，但是当数据量较大，或符合一定的规范时，我们可以使用数据库来进行数据的存储。

在浏览器提供的数据库中，共有`web sql`和`IndexedDB`两种。相较于HTML5已经废弃的`web sql`来说，更推荐大家使用`IndexedDB`。

# 结构

下面，我们通过一张图来了解下，IndexedDB整体的结构。

![clipboard.png](https://segmentfault.com/img/bV4R5H)

类比`sql`型数据库，`IndexedDB`中的DB（数据库）就是`sql`中的DB，而`Object Store(存储空间)`则是`数据表`,`Item`则等于表中的一条`记录`。

# 使用IndexedDB

现在，我们将其根据`IndexedDB`的结构来对其操作进行介绍，能让大家对这个存储空间有一个初步的了解。我们主要介绍：

- 数据库操作
- 数据表操作
- 数据操作

## 数据库操作

### 创建或打开数据库

使用`IndexedDB`第一步，就是创建或打开一个数据库。我们使用`window.indexedDB.open(DBName)`这个API来打进行操作。具体示例如下：

```javascript
const request = window.indexedDB.open('test');

request.onupgradeneeded = function (event) {
    
}

request.onsuccess = function(event) {
	//request === event.target;
}
request.onerror = function(event) {}
```

调用此接口时，如果当前数据库不存在，则会创建一个新的数据库。

当数据库建立连接时，会返回一个`IDBOpenDBRequest`对象。

在连接建立成功时，会触发`onsuccess`事件，其中函数参数`event`的`target`属性就是`request`对象。

而在数据库创建或者版本更新时，会触发`onupgradeneeded`事件。

### 更新数据库版本号

`window.indexedDB.open`的第二个参数即为版本号。在不指定的情况下，默认版本号为1。具体示例如下:

```javascript
const request = window.indexedDB.open('test', 2);
```

在需要更新数据库的`schema(模式)`时，需要更新版本号。此时我们指定一个高于之前版本的版本号，就会触发`onupgradeneeded`事件。类似的，当此数据库不存在时，也会触发此事件并且将版本更新到置顶版本。

我们需要注意的是，版本号是一个`Unsigned long long`数字，这意味着它可以是一个非常大的整数。但是，它不能是一个小数，否则它将会被转为最近的整数，同时有可能导致`onUpgradeneeded`事件不触发(bug)。

## 存储空间操作

### 创建存储空间

我们使用`createObjectStore`来创建一个存储空间。同时，使用`createIndex`来创建它的索引。具体示例如下：

```javascript
var request = window.indexedDB.open('test', 1);

request.onupgradeneeded = function (event) {
    var db = event.target.result;
    var objectStore = db.createObjectStore('table1', {keyPath: 'id', autoIncrement: true});

    objectStore.createIndex('name', 'name', {unique: false});
}

request.onerror = function (event) {
    alert("Why didn't you allow my web app to use IndexedDB?!");
};
```

**注：只能在`onupgradeneeded`回调函数中创建存储空间，而不能在数据库打开后的`success`回调函数中创建。**

通过`createObjectStore`能够创建一个存储空间。接受两个参数：

1. 第一个参数，存储空间的名称，即我们上面的`customers`。
2. 第二个参数，指定存储的`keyPath`值为存储对象的某个属性，这个属性能够在获取存储空间数据的时候当做key值使用。`autoIncrement`指定了`key`值是否自增（当key值为默认的从1开始到2^53的整数时）。

而`createIndex`能够给当前的存储空间设置一个索引。它接受三个参数：

1. 第一个参数，索引的名称。
2. 第二个参数，指定根据存储数据的哪一个属性来构建索引。
3. 第三个属性， options对象，其中属性`unique`的值为`true`表示不允许索引值相等。

## 数据操作

### 事务

在`IndexedDB`中，我们也能够使用事务来进行数据库的操作。事务有三个模式（常量已经弃用）：

- `readOnly`，只读。
- `readwrite`，读写。
- `versionchange`，数据库版本变化。

我们创建一个事务时，需要从上面选择一种模式，如果不指定的话，则默认为只读模式。具体示例如下：

```javascript
const transaction = db.transaction(['customers'], 'readwrite');
```

事务函数`transaction`的第一个参数为需要关联的存储空间，第二个可选参数为事务模式。与上面类似，事务成功时也会触发`onsuccess`函数，失败时触发`onerror`函数。

事务的操作都是原子性的。

### 增加数据

当存储空间初始化完成后，我们可以把数据放入存储空间中。直接调用`add`方法就可以将数据放入存储空间内，具体示例如下：

```javascript
var request = window.indexedDB.open('test', 1);

request.onsuccess = function (event) {
    var db = event.target.result;

    var transaction = db.transaction(['table1'], 'readwrite');

    var objectStore = transaction.objectStore('table1');

    var index = objectStore.index('name');

    objectStore.add({name: 'a', age: 10});
    objectStore.add({name: 'b', age: 20});
}
```

注：`add`方法中的第二个参数key值是指定存储空间中的`keyPath`值，如果`data`中包含`keyPath`值或者此值为自增值，那么可以略去此参数。

### 查找数据

#### 通过特定值获取数据

当我们需要从存储空间获取数据时，我们可以通过以下的方法：

```javascript
var request = window.indexedDB.open('test', 1);

request.onsuccess = function (event) {
    var db = event.target.result;

    var transaction = db.transaction(['table1'], 'readwrite');

    var objectStore = transaction.objectStore('table1');

    var request = objectStore.get(1);

    request.onsuccess = function (event) {
        // 对 request.result 做些操作！
        console.log(request.result);
    };

    request.onerror = function (event) {
        // 错误处理!
    };
}
```

#### 通过游标获取数据

当你需要便利整个存储空间中的数据时，你就需要使用到游标。游标使用方法如下：

```javascript
var request = window.indexedDB.open('test', 1);

request.onsuccess = function (event) {
    var db = event.target.result;

    var transaction = db.transaction(['table1'], 'readwrite');

    var objectStore = transaction.objectStore('table1');

    var request = objectStore.openCursor();

    request.onsuccess = function (event) {
        var cursor = event.target.result;
        if (cursor) {
            // 使用Object.assign方法是为了避免控制台打印时出错
            console.log(Object.assign(cursor.value));
            cursor.continue();
        }
    };

    request.onerror = function (event) {
        // 错误处理!
    };
}
```

使用游标时有一个需要注意的地方，当游标便利整个存储空间但是并未找到给定条件的值时，仍然会触发`onsuccess`函数。

`openCursor`和`openKeyCursor`有两个参数：

1. 第一个参数，遍历范围，指定游标的访问范围。该范围通过一个`IDBKeyRange`参数的方法来获取。

   遍历范围参数具体示例如下：

   ```javascript
   // 匹配值 key === 1
   const singleKeyRange = IDBKeyRange.only(1);

   // 匹配值 key >= 1
   const lowerBoundKeyRange = IDBKeyRange.lowerBound(1);

   // 匹配值 key > 1
   const lowerBoundOpenKeyRange = IDBKeyRange.lowerBound(1, true);

   // 匹配值 key < 2
   const upperBoundOpenKeyRange = IDBKeyRange.upperBound(2, true);

   // 匹配值 key >= 1 && key < 2
   const boundKeyRange = IDBKeyRange.bound(1, 2, false, true);

   index.openCursor(boundKeyRange).onsuccess = function(event) {
     const cursor = event.target.result;
     if (cursor) {
       // Do something with the matches.
       cursor.continue();
     }
   };
   ```

   ​

2. 第二个参数，便利顺序，指定游标便利时的顺序和处理相同id（keyPath属性指定字段）重复时的处理方法。改范围通过特定的字符串（`IDBCursor`的常量已经弃用）来获取。其中：

   - `next`，从前往后获取所有数据（包括重复数据）
   - `prev`，从后往前获取所有数据（包括重复数据）
   - `nextunique`，从前往后获取数据（重复数据只取第一条，索引重复即认为重复，下同）
   - `prevunique`，从后往前获取数据（重复数据只取第一条）

   遍历顺序参数具体示例如下：

   ```javascript
   var request = window.indexedDB.open('test', 1);

   request.onsuccess = function (event) {
       var db = event.target.result;

       var transaction = db.transaction(['table1'], 'readwrite');

       var objectStore = transaction.objectStore('table1');

       var lowerBoundOpenKeyRange = IDBKeyRange.lowerBound(1, false);
       var request = objectStore.openCursor(lowerBoundOpenKeyRange, IDBCursor.PREV);

       request.onsuccess = function (event) {
           var cursor = event.target.result;
           if (cursor) {
               // 使用Object.assign方法是为了避免控制台打印时出错
               console.log(Object.assign(cursor.value));
               cursor.continue();
           }
       };

       request.onerror = function (event) {
           // 错误处理!
       };
   }
   ```

#### 使用索引

在前面构建数据库时，我们创建了两个索引。现在我们也可以通过索引来进行数据检索。他的本质还是通过之前获取数据的API来进行，只是将原来使用的`keyPath`属性转换成为了索引指定的属性。具体示例如下：

```javascript
var request = window.indexedDB.open('test', 1);

request.onsuccess = function (event) {
    var db = event.target.result;

    var transaction = db.transaction(['table1'], 'readwrite');

    var objectStore = transaction.objectStore('table1');

    var index = objectStore.index('name');

    // 第一种，get方法
    index.get('a').onsuccess = function (event) {
        console.log(event.target.result);
    }

    // 第二种，普通游标方法
    index.openCursor().onsuccess = function (event) {
        console.log('openCursor:', event.target.result.value);
    }

    // 第三种，键游标方法，该方法与第二种的差别为：普通游标带有value值表示获取的数据，而键游标没有
    index.openKeyCursor().onsuccess = function (event) {
        console.log('openKeyCursor:', event.target.result);
    }
}
```

### 修改数据

当需要修改存储空间中的数据时，我们可以使用以下的API：

```javascript
var objectStore = transaction.objectStore("customers");

var request = objectStore.put(data);

request.onsuccess = function (event) {
    
}
```

注：`put`方法不仅能够修改现有数据，也能够往存储空间中增加新的数据。

### 删除数据

当我们需要删除已经无用的数据时，我们可以通过以下方法：

```javascript
var objectStore = transaction.objectStore("customers");

var request = objectStore.delete(name);

request.onsuccess = function (event) {
    
}
```

## 异常处理

在浏览器有如下操作的情况下，indexedDB可能会出现异常：

- 用户清除浏览器缓存
- 存储空间超过大小限制

此时，需要对错误进行捕获，并且对用户进行提示。此章节不是本文重点，再此略过。

# 扩展须知

## 取值相关

### key值能够接受的数据类型

在`IndexedDB`中，键值对中的`key`值可以接受以下几种类型的值：

- number
- data
- string
- binary
- array

具体说明可以见文档[此处](https://w3c.github.io/IndexedDB/#key-construct)。

### key path能够接受的数据类型

当一个`key`值变为主键，即`keyPath`时，它的值就只能是以下几种：

- Blob
- File
- Array
- String

**注：空格不能出现在key path中**。

具体说明可以见文档[此处](https://w3c.github.io/IndexedDB/#key-path-construct)。

### value能够接受的数据类型

在`IndexedDB`中，value能够接受ECMA-262中所有的类型的值，例如String，Date，ImageDate等。

## 事务相关

### 事务中断后，会不会影响key值的自增

`IndexedDB`在没有指定key值的时候就会采用自增的key值。如果一个事务在中途中断，那么key值的自增将会从中断的事务开始前的key开始。

## 安全相关

`IndexedDB`也受到浏览器[同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)的限制。

## 用户相关

### 清空缓存

用户在清除浏览器缓存时，可能会清除`IndexedDB`中相关的数据。

### 访问权限

部分浏览器如Safari手机版隐私模式在访问`IndexedDB`时，可能会出现由于没有权限而导致的异常（LocalStorage也会），需要进行异常处理。

# 总结

`IndexedDB`在本地存储中有着无可替代的作用，是替代关系型数据库`web sql`的产品，能够对大量数据进行存储。在许多需要运用离线存储的场景下，它能够给我们提供有效的支撑。

但是，`IndexedDB`在使用过程中仍然需要避免可能会出现的一些问题，或者对可能导致的不利影响有一定的容错处理。这样才不会对应用产生重大影响。

# 参考文献
- [浏览器的同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)
- [使用indexedDB MDN入门](https://developer.mozilla.org/zh-CN/docs/Web/API/IndexedDB_API/Using_IndexedDB)
- [IndexedDB API参考](https://developer.mozilla.org/zh-CN/docs/Web/API/IndexedDB_API)
- [W3C IndexedDB 2.0规范](https://w3c.github.io/IndexedDB/)