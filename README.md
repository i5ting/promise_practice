promise_practice
================


## nodejs

目前nodejs了分2类

- 传统的
- 基于generator的

可选的promise/A+实现

- async
- eventproxy（国人作品，基于事件的这个想法很赞，cnodejs是基于它的）
- then.js（国人作品，还不错，没经过测试，不太敢用）
- bluebird
- q（非常早得实现，2种都支持，很完善，但效率有点低）
- co（只是支持generator）
- deferred(https://github.com/medikoo/deferred)

这里面我最喜欢的是bluebird


### 场景


#### 异步编程中的错误处理

人性的、理想的也正如很多编程语言中已经实现的错误处理方式应该是这样：

	try {
	    var val = JSON.parse(fs.readFileSync("file.json"));
	}catch(SyntaxError e) {//json语法错误
	    console.error("不符合json格式");
	}catch(Error e) {//其它类型错误
	    console.error("无法读取文件")
	}


### promisify

下面我们看如何对fs.readFileAsync方法进行promisify，依然是使用bluebird。

	var Promise = require('bluebird')
	fs.readFileAsync = Promise.promisify(fs.readFie, fs)

怎么样，就是如此简单！对于bluebird它还有一个更强大的方法，那就是promisify的高级版本 promisifyAll，比如：


	var Promise = require('bluebird')
	Promise.promisifyAll(fs)

执行完上面的代码之后，fs对象下所有的异步方法都会对应的生成一个Promise版本方法，比如fs.readFile对应fs.readFileAsync，fs.mkdir对应fs.mkdirAsync，以此类推。

另外要注意的就是，Promise版本的函数除了最后一个参数（回调函数），其它参数与原函数均一一对应，调用的时候别忘了传递原有的参数。

对fs的promisification还不能令我满足，我需要更神奇的魔法：

	// redis
	var Promise = require("bluebird");
	Promise.promisifyAll(require("redis"));
 
	// mongoose
	var Promise = require("bluebird");
	Promise.promisifyAll(require("mongoose"));
 
	// mongodb
	var Promise = require("bluebird");
	Promise.promisifyAll(require("mongodb"));
 
	// mysql
	var Promise = require("bluebird");
	Promise.promisifyAll(require("mysql/lib/Connection").prototype);
	Promise.promisifyAll(require("mysql/lib/Pool").prototype);
 
	// request
	var Promise = require("bluebird");
	Promise.promisifyAll(require("request"));
 
	// mkdir
	var Promise = require("bluebird");
	Promise.promisifyAll(require("mkdirp"));
 
	// winston
	var Promise = require("bluebird");
	Promise.promisifyAll(require("winston"));
 
	// Nodemailer
	var Promise = require("bluebird");
	Promise.promisifyAll(require("nodemailer"));
 
	// pg
	var Promise = require("bluebird");
	Promise.promisifyAll(require("pg"));


源码


```
function promisify(callback, receiver) {
    return makeNodePromisified(callback, receiver, undefined, callback);
}

Promise.promisify = function (fn, receiver) {
    if (typeof fn !== "function") {
        throw new TypeError("fn must be a function");
    }
    if (isPromisified(fn)) {
        return fn;
    }
    return promisify(fn, arguments.length < 2 ? THIS : receiver);
};
```

所以核心是`makeNodePromisified`

另外

```
function promisifyAll(obj, suffix, filter, promisifier) {
    var suffixRegexp = new RegExp(escapeIdentRegex(suffix) + "$");
    var methods =
        promisifiableMethods(obj, suffix, suffixRegexp, filter);

    for (var i = 0, len = methods.length; i < len; i+= 2) {
        var key = methods[i];
        var fn = methods[i+1];
        var promisifiedKey = key + suffix;
        obj[promisifiedKey] = promisifier === makeNodePromisified
                ? makeNodePromisified(key, THIS, key, fn, suffix)
                : promisifier(fn);
    }
    util.toFastProperties(obj);
    return obj;
}
```

```
function makeNodePromisifiedClosure(callback, receiver) {
    function promisified() {
        var _receiver = receiver;
        if (receiver === THIS) _receiver = this;
        if (typeof callback === "string") {
            callback = _receiver[callback];
        }
        var promise = new Promise(INTERNAL);
        promise._setTrace(undefined);
        var fn = nodebackForPromise(promise);
        try {
            callback.apply(_receiver, withAppended(arguments, fn));
        } catch(e) {
            var wrapped = maybeWrapAsError(e);
            promise._attachExtraTrace(wrapped);
            promise._reject(wrapped);
        }
        return promise;
    }
    promisified.__isPromisified__ = true;
    return promisified;
}
```

如果方法没有promise，就加上，加上的方法和angularjs的ioc是一样的。都是通过把方法转成字符串，然后注入一部分代码
使之完成某些功能。

东西虽好，可不要多用哦，效率会损失很多的。

## oc

### 推荐

http://promisekit.org/


