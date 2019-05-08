---
title: Async 库之 series 源码分析
date: 2019-04-22 12:39:04
categories:
- Nodejs
tags:
- Nodejs
---

&emsp;&emsp;学习过Nodejs的同学应该少不了会遇到Async库，其中的series方法应该有人用过，好奇心作怪就做了一个测试案例来分析下内部实现。先上测试代码，其中包括同步任务及异步任务（该case基于Async 2.6.2 做的分析）：

   ``` javascript 
    var async = require('async');

    var test = function(){
        return new Promise(function(resovle,reject){
            setTimeout(function() {
                resovle(1);
            }, 200);
        });
    }

    async.series({
        one: async function(callback) { //此处callback无效

            var result = await test();
            return result;
            // callback(null,result);
        },
        two: function(callback){
            setTimeout(function() {
                callback(null, 2);
            }, 100);
        }
    }, function(err, results) {
        // results is now equal to: {one: 1, two: 2}
        console.log(results);
    });
    ```
点击series进入到源码，看到series方法调用的是 _parallel 方法，之后调用了很多高阶函数，容易晕。先把重要代码贴上：
   ``` javascript 
    var eachOfSeries = doLimit(eachOfLimit, 1);
   
    function series(tasks, callback) {
       _parallel(eachOfSeries, tasks, callback);
    }
    
    function _parallel(eachfn, tasks, callback) {
        callback = callback || noop;
        var results = isArrayLike(tasks) ? [] : {};

        eachfn(tasks, function (task, key, callback) {
            wrapAsync(task)(function (err, result) {
                if (arguments.length > 2) {
                    result = slice(arguments, 1);
                }
                results[key] = result;
                callback(err);
            });
        }, function (err) {
            callback(err, results);
        });
    }

    function doLimit(fn, limit) {
        return function (iterable, iteratee, callback) {
            return fn(iterable, limit, iteratee, callback);
        };
    }
    function eachOfLimit(coll, limit, iteratee, callback) {
        _eachOfLimit(limit)(coll, wrapAsync(iteratee), callback);
    }
    ```
首先分析主函数 _parallel，该方法主要部分是 eachfn，而eachfn实际上是 doLimit，再进入到 doLimit 后发现，实际就变成了执行  _eachOfLimit 内部函数，最终看到了执行的地方：
   ``` javascript 
    function _eachOfLimit(limit) {
        return function (obj, iteratee, callback) {
            callback = once(callback || noop);
            if (limit <= 0 || !obj) {
                return callback(null);
            }
            var nextElem = iterator(obj);
            var done = false;
            var running = 0;     // 记录当前正在执行的任务数
            var looping = false; // 标记任务在进入异步队列前的同步过程，执行中或结束

            function iterateeCallback(err, value) {
                running -= 1;   // 每执行完成一个任务就减一 
                if (err) {      // 出错则返回
                    done = true;
                    callback(err);
                }
                // 这里分两种情况：
                // 1. 跳出循环，即 value = {}
                // 2. 所有任务已全部执行完成
                else if (value === breakLoop || (done && running <= 0)) {
                    done = true;
                    return callback(null);
                }
                else if (!looping) {
                    replenish();
                }
            }

            function replenish () {
                looping = true;   // 同步过程执行中
                // 若仍有未执行的任务 并且 当前执行的任务数小于上限 limit = 1，即当前只有一个任务在执行
                // 所以这里就确保了任务的顺序执行，执行完成后 looping = false，回调后继续调用 replenish 执行下一个任务
                while (running < limit && !done) { 
                    var elem = nextElem();
                    if (elem === null) {
                        done = true;        // 任务已全部执行，等待所有任务完成
                        if (running <= 0) { // 若任务已全部执行完成则返回
                            callback(null);
                        }
                        return;
                    }
                    running += 1; // 每执行一个任务就加一
                    iteratee(elem.value, elem.key, onlyOnce(iterateeCallback));
                }
                looping = false; // 同步过程已结束，等异步通知 iterateeCallback
            }

            replenish();
        };
    }
   ```
通过代码可以知道该方法是通过设置当前执行函数的个数来保证任务的顺序执行，即 limit=1。待前一个任务执行完成并通过callback返回后才执行下一个函数。该方法使用了一个running变量来标记当前正在执行的任务，每执行一个就加一，待回调完成后减一。

该方法的执行过程会涉及到几个问题：
- *** 任务会分为同步任务及异步任务，如何保证他们正确顺序执行？***
 
- *** 假如series的任务参数是es6中迭代器对象，如何任务正确迭代遍历？***

*** 1 任务会分为同步任务及异步任务，如何保证他们正确顺序执行？***

&emsp;&emsp;同步任务很好理解，函数执行完成后直接调用callback函数即可返回，然后继续执行下一个任务。但是异步任务就有可能是一个Promise或者加了async关键字的函数.这时就需要判断一个函数是否是异步函数。单单判断函数异步是异步函数还不行，因为异步函数不会自动执行，所以还要异步函数可以自动执行然后顺利回调callback。
&emsp;&emsp;** 1.1 &emsp;判断异步函数 **
&emsp;&emsp;查看源码可以看到：

   ``` javascript
   var supportsSymbol = typeof Symbol === 'function';
   
   function isAsync(fn) {
       return supportsSymbol && fn[Symbol.toStringTag] === 'AsyncFunction';
   }
   ```

   &emsp;&emsp;ES6 引入了一种新的原始数据类型`Symbol`，表示独一无二的值。它是 JavaScript 语言的第七种数据类型，前六种是：`undefined`、`null`、布尔值（Boolean）、字符串（String）、数值（Number）、对象（Object）。Symbol 值通过 ** Symbol 函数 ** 生成。因为需要用到Symbol做判断，所以首先判断是否支持该原始类型。

   &emsp;&emsp;对象的Symbol.toStringTag属性，指向一个方法。在该对象上面调用Object.prototype.toString方法时，如果这个属性存在，它的返回值会出现在toString方法返回的字符串之中，表示对象的类型.
   ``` javascript
    // 例一
    ({[Symbol.toStringTag]: 'Foo'}.toString())
    // "[object Foo]"

    // 例二
    class Collection {
      get [Symbol.toStringTag]() {
        return 'xxx';
      }
    }
    let x = new Collection();
    Object.prototype.toString.call(x) // "[object xxx]"
   ```
   通过阮大师的es了解到了以上情况，但是fn[Symbol.toStringTag] 为什么会是 'AsyncFunction'这个没理解。估计在内部做了封装。判断是否是异步函数还可以通过以下方式来判断（generator函数同理）。
``` javascript
    const AsyncFunction = (async () => {}).constructor;

    asyncFn instanceof AsyncFunction  // true
    ```
&emsp;&emsp;** 1.2 &emsp;执行异步函数 **
&emsp;&emsp;通过以下代码可知，若当前函数是异步函数则将该函数做异步包装，通过func.apply(this, args)执行函数，若该函数返回的是一个Promise对象，则执行then函数获取执行结果，然后执行callback。
``` javascript
    function wrapAsync(asyncFn) {
        return isAsync(asyncFn) ? asyncify(asyncFn) : asyncFn;
    }
    function asyncify(func) {
        return initialParams(function (args, callback) {
            var result;
            try {
                result = func.apply(this, args);
            } catch (e) {
                return callback(e);
            }
            // if result is Promise object
            if (isObject(result) && typeof result.then === 'function') {
                result.then(function(value) {
                    invokeCallback(callback, null, value);
                }, function(err) {
                    invokeCallback(callback, err.message ? err : new Error(err));
                });
            } else {
                callback(null, result);
            }
        });
    }
    var initialParams = function (fn) {
        return function (/*...args, callback*/) {
            var args = slice(arguments);
            var callback = args.pop();
            fn.call(this, args, callback);
        };
    };
    function invokeCallback(callback, error, value) {
        try {
            callback(error, value);
        } catch (e) {
            setImmediate$1(rethrow, e);
        }
    }
  
```
执行callback时还加了一个try catch，若callback出现异常则通过异步的方式抛出该异常，可以看看这段代码。最先通过setImmediate方法抛出异常，若该方法没有则使用 process.nextTick ，若再没有则执行通过 setTimeout 方法抛出。
``` javascript 
    var hasSetImmediate = typeof setImmediate === 'function' && setImmediate;
    var hasNextTick = typeof process === 'object' && typeof process.nextTick === 'function';

    function fallback(fn) {
        setTimeout(fn, 0);
    }

    function wrap(defer) {
        return function (fn/*, ...args*/) {
            var args = slice(arguments, 1);
            defer(function () {
                fn.apply(null, args);
            });
        };
    }

    var _defer;

    if (hasSetImmediate) {
        _defer = setImmediate;
    } else if (hasNextTick) {
        _defer = process.nextTick;
    } else {
        _defer = fallback;
    }

    var setImmediate$1 = wrap(_defer);
```
看过Promise源码的应该可以知道Promise也是使用该方式来保证任务的异步执行，只是没有用到 setTimeout 方法。代码注释里说到为什么要首先使用setImmediate，是因为process.nextTick无法处理递归调用。对这块还不太理解，小伙伴们知道的麻烦告知一声。
![](./20190420182907.png)

*** 2 假如series的任务参数是es6中迭代器对象，如何任务正确迭代遍历？***
&emsp;&emsp;在es6之前我们传入的参数可以是一个数组或者json对象，但在es6增加了一个生成器函数。生成器函数会返回一个迭代器对象。默认情况下定义的对象（object）是不可迭代的，但是可以通过Symbol.iterator创建迭代器。如：
``` javascript
    const list = {
        items:[5,9,18],
        *[Symbol.iterator](){
            for(let item of this.items){
                yield item;
            }
        }
    }
    
    for(let item of list){
       console.log(item); //5,9,18
    }
```

所以series的入参也就可以是一个可迭代的对象。测试代码修改如下：
``` javascript 
    // 将我们的任务直接放到items数组中，items这个属性可以自行定义，只要保证正确遍历即可
    const list = {
        items:[async function(callback) { //此处callback无效

            var result = await test();
            return result;
            // callback(null,result);
        },function(callback){
            setTimeout(function() {
                callback(null, 2);
            }, 100);
        }],
        *[Symbol.iterator](){
            for(let item of this.items){
                yield item;
            }
        }
    }
    async.series(list, function(err, results) {
        // results is now equal to: {one: 1, two: 2}
        console.log(results);
    });
```
代码分析到这里，应该就可以想到series的入参是需要判断的。该参数很有可能是一个迭代器对象。通过源码可以发下这里面做了判断：

``` javascript
    // _eachOfLimit 方法中的( async 2.6.2 )：
    //   var nextElem = iterator(obj); // line 979
    function iterator(coll) {
        if (isArrayLike(coll)) {
            return createArrayIterator(coll);
        }

        var iterator = getIterator(coll);
        return iterator ? createES2015Iterator(iterator) : createObjectIterator(coll);
    }
```
![](./20190420174458.png)

![](./20190420174957.png)

针对不同的入参就需要不同的遍历方式：
- **若是数组则通过下标进行遍历**

- **若是生成器返回的迭代器对象则通过next()的方式进行遍历**

- **若只是一个对象，则通过遍历其所有的key进行遍历**

数组遍历的方式就不用说了，咱们来看看其他两种方式
**2.1 &emsp;通过生成器返回的迭代器进行遍历**
&emsp;&emsp;该迭代器是通过next()方法来获得当前对象，其中包含done（是否遍历完成），value（当前元素）。学过Java或C#应该容易理解，他们都有一个hasNext()及next()方法，hasNext()用于判断是否还有下一个元素，next()用于获取下一个元素。生成器generator是通过协程的方式实现，这个还需深入了解。
``` javascript
  function createES2015Iterator(iterator) {
        var i = -1;
        return function next() {
            var item = iterator.next();
            if (item.done)
                return null;
            i++;
            return {value: item.value, key: i};
        }
    }
```

**2.2 &emsp;通过遍历所有的key进行遍历**

``` javascript 
  function createObjectIterator(obj) {
        var okeys = keys(obj);
        var i = -1;
        var len = okeys.length;
        return function next() {
            var key = okeys[++i];
            return i < len ? {value: obj[key], key: key} : null;
        };
    }
```
该方式的重点在keys方法，该遍历方式远比我想象的要复杂些。
``` javascript 
    function keys(object) {
      return isArrayLike(object) ? arrayLikeKeys(object) : baseKeys(object);
    }
    function arrayLikeKeys(value, inherited) {
      var isArr = isArray(value),
          isArg = !isArr && isArguments(value),
          isBuff = !isArr && !isArg && isBuffer(value),
          isType = !isArr && !isArg && !isBuff && isTypedArray(value),
          skipIndexes = isArr || isArg || isBuff || isType,
          result = skipIndexes ? baseTimes(value.length, String) : [],
          length = result.length;

      for (var key in value) {
        if ((inherited || hasOwnProperty$1.call(value, key)) &&
            !(skipIndexes && (
               // Safari 9 has enumerable `arguments.length` in strict mode.
               key == 'length' ||
               // Node.js 0.10 has enumerable non-index properties on buffers.
               (isBuff && (key == 'offset' || key == 'parent')) ||
               // PhantomJS 2 has enumerable non-index properties on typed arrays.
               (isType && (key == 'buffer' || key == 'byteLength' || key == 'byteOffset')) ||
               // Skip index properties.
               isIndex(key, length)
            ))) {
          result.push(key);
        }
      }
      return result;
    }

    function baseKeys(object) {
      if (!isPrototype(object)) {
        return nativeKeys(object);
      }
      var result = [];
      for (var key in Object(object)) {
        if (hasOwnProperty$3.call(object, key) && key != 'constructor') {
          result.push(key);
        }
      }
      return result;
    }
```
keys方法会使用 isArrayLike 方法先判断当前对象是否是类似数组的对象，即判断
- ** 该对象不是函数，该判断有些复杂，还考虑了浏览器兼容问题** 
- ** 该对象拥有value.length属性，并且该值是一个大于-1且小于MAX_SAFE_INTEGER的整数 ** 
  注意这里必须是整数，这里有一个巧妙的设计，即： value %  1== 0。可以拿一个小数 2.4 验证一下
  ![](./20190420195011.png)
  
  
``` javascript 
    /**
     * Checks if `value` is array-like. A value is considered array-like if it's
     * not a function and has a `value.length` that's an integer greater than or
     * equal to `0` and less than or equal to `Number.MAX_SAFE_INTEGER`.
     *
     * @static
     * @memberOf _
     * @since 4.0.0
     * @category Lang
     * @param {*} value The value to check.
     * @returns {boolean} Returns `true` if `value` is array-like, else `false`.
     * @example
     *
     * _.isArrayLike([1, 2, 3]);
     * // => true
     *
     * _.isArrayLike(document.body.children);
     * // => true
     *
     * _.isArrayLike('abc');
     * // => true
     *
     * _.isArrayLike(_.noop);
     * // => false
     */
    function isArrayLike(value) {
      return value != null && isLength(value.length) && !isFunction(value);
    }
    
    function isLength(value) {
      return typeof value == 'number' &&
        value > -1 && value % 1 == 0 && value <= MAX_SAFE_INTEGER;
    }
    
    /** `Object#toString` result references. */
    var asyncTag = '[object AsyncFunction]';
    var funcTag = '[object Function]';
    var genTag = '[object GeneratorFunction]';
    var proxyTag = '[object Proxy]';
    
    function isFunction(value) {
      if (!isObject(value)) {
        return false;
      }
      // The use of `Object#toString` avoids issues with the `typeof` operator
      // in Safari 9 which returns 'object' for typed arrays and other constructors.
      var tag = baseGetTag(value);
      return tag == funcTag || tag == genTag || tag == asyncTag || tag == proxyTag;
    }
    
    function isObject(value) {
      var type = typeof value;
      return value != null && (type == 'object' || type == 'function');
    }
```
本来想弄一个测试案例来试试 arrayLikeKeys 方法，但是不太好举例，因为在 iterator 方法中已经做了一次 isArrayLike 的判断，所以按照给方法进去的话应该就不会在走 arrayLikeKeys 方法。arrayLikeKeys 这块的实现有时间可以研究下，这里就做了其他的案例测试。
&emsp;&emsp;将我传入的参数改为如下 obj 对象，执行会报错，因为有两个属性不是函数。所以执行的时候会出错
![](./20190420193031.png)

series实现暂且分析到这里，通过源码分析可以发现一些有趣的实现，比如如何确保函数只会执行一次，有两个方法实现一个是once，一个是onlyOnce， 一目了然。他们都是使用了高阶函数，然后将函数复制一份后赋值 null ：
``` javascript
function once(fn) {
    return function () {
        if (fn === null) return;
        var callFn = fn;
        fn = null;
        callFn.apply(this, arguments);
    };
}

function onlyOnce(fn) {
    return function() {
        if (fn === null) throw new Error("Callback was already called.");
        var callFn = fn;
        fn = null;
        callFn.apply(this, arguments);
    };
}
```



