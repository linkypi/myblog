---
title: Async 库之 parallel 源码分析
date: 2019-04-22 12:39:04
categories:
- Nodejs
tags:
- Nodejs
---

&emsp;&emsp;前面做过 series 的源码分析后，大体知道了其内部原理。现在来看看 parallel 实现。首先贴出重要代码部分（该case基于Async 2.6.2 做的分析）：
``` javascript 
    exports.parallel = parallelLimit;
    function parallelLimit(tasks, callback) {
        _parallel(eachOf, tasks, callback);
    }
    
    // 与 series 调用同一个方法
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

    var eachOf = function(coll, iteratee, callback) {
        // 根据任务的不同返回不同实现，在 series 文章中分析过 isArrayLike 
        var eachOfImplementation = isArrayLike(coll) ? eachOfArrayLike : eachOfGeneric;
        eachOfImplementation(coll, wrapAsync(iteratee), callback);
    };
    
    // eachOf implementation optimized for array-likes
    function eachOfArrayLike(coll, iteratee, callback) {
        callback = once(callback || noop);
        var index = 0,
            completed = 0,
            length = coll.length;
        if (length === 0) {
            callback(null);
        }

        function iteratorCallback(err, value) {
            if (err) {
                callback(err);
            } else if ((++completed === length) || value === breakLoop) {
                callback(null);
            }
        }

        for (; index < length; index++) {
            iteratee(coll[index], index, onlyOnce(iteratorCallback));
        }
    }
    
    var eachOfGeneric = doLimit(eachOfLimit, Infinity);
    
```
这里涉及到两个实现：
- ** eachOfArrayLike **
- ** eachOfGeneric ** 

***  1.  &emsp; eachOfGeneric  ***
&emsp;&emsp; 先来分析使用 eachOfGeneric 的情况，实际 eachOfGeneric 就是 doLimit。只不过与 series 方法对比的话，这里的 limit 参数是一个无穷大的数（即当前执行任务的数量没有上限），而 series 的 limit 参数是 1，因为 series 需要保证顺序执行。 还是回到具体实现的代码：
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
            var looping = false;

            function iterateeCallback(err, value) {
                running -= 1;   // 每执行完成一个任务就减一 
                if (err) {      // 出错则返回
                    done = true;
                    callback(err);
                }
                // 这里分两种情况：
                // 1. 跳出循环，即 value = {}
                // 2. 所有任务已全部执行完成，正是这里确保了所有任务执行完成后统一返回执行结果
                else if (value === breakLoop || (done && running <= 0)) {
                    done = true;
                    return callback(null);
                }
                else if (!looping) { 
                    // 此处不会响应，因为在 replenish 中会不断的执行任务，直到没有任务
                    // 也就是 elem = null 时就已经 return 返回，也就不会执行  looping = false
                    replenish();
                }
            }

            function replenish () {
                looping = true;
                while (running < limit && !done) { // 还有未执行的任务 并且 当前执行的任务数小于上限，limit = Infinity
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
                looping = false;
            }

            replenish();
        };
    }
```
该代码可以用这个入参做测试：
``` javascript 
const list = {
    items:[
        async function(callback) { //此处callback无效
            var result = await test();
            return result;
            // callback(null,result);
        },function(callback){
            setTimeout(function() {
                callback(null, 2);
            }, 100);
        },
        function(callback){
            setTimeout(function() {
                callback(null, 12);
            }, 10000);
        }
    ],
    *[Symbol.iterator](){
        for(let item of this.items){
            yield item;
        }
    }
};
```
***  2.  &emsp; eachOfArrayLike  ***
下面来看看 eachOfArrayLike 的实现：
``` javascript 
// eachOf implementation optimized for array-likes
function eachOfArrayLike(coll, iteratee, callback) {
    callback = once(callback || noop);
    var index = 0,
        completed = 0,
        length = coll.length;
    if (length === 0) {
        callback(null);
    }

    function iteratorCallback(err, value) {
        if (err) {
            callback(err);
        } else if ((++completed === length) || value === breakLoop) {
            callback(null);
        }
    }

    for (; index < length; index++) {
        iteratee(coll[index], index, onlyOnce(iteratorCallback));
    }
}
```
入参可以随便创建一个任务数组即可：

``` javascript 
    var ar = [
        async function(callback) { //此处callback无效
            var result = await test();
            return result;
            // callback(null,result);
        },
        function(callback){
            setTimeout(function() {
                callback(null, 2);
            }, 100);
        },
        function(callback){
            setTimeout(function() {
                callback(null, 12);
            }, 10000);
        }
    ];
```

这里不太明白为什么还要做 isArrayLike 区分，因为在 doLimit中已经做了区分。我将源码改为统一的 eachOfGeneric实现，然后使用任务数组做入参，结果并没有什么不同。




