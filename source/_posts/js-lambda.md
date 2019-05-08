---
title: Javascript Lambda的简单实现
date: 2017-03-19
categories:
- javascript
- lambda
tags:
- javascript
- lambda
---

<center>**由来**</center>

今早面试看到一题是说使用js实现.net中的where功能，当时脑子里一片空白，只能回来脑补下（本来想将博客放到github及Blogger上，但限于配置麻烦，暂且放这里）。真心不知道ES6以下的js支不支持 " => "， 后来在chrome上测试下竟然支持，我没有打开chrome对ES6的支持。这是一个简单实现，主要使用到了eval，暂时没有考虑其性能，同时实现了 select 及distinct方法，欢迎大牛指正。既然语法支持那就可以对传入的参数做进一步处理，首先将参数转为字符串然后使用正则提取变量及表达式，到使用eval执行语句时再将表达式中的参数替换为指定的值。以下代码还可以重构，可以考虑写一个方法专门解析表达式、增加表达式正确性校验等等。

``` javascript
        Array.prototype.where = function(){
            var args = arguments[0].toString();
            var matches = args.match(/(\w)(\s+)?=>(.*)+/);
            if(!matches){
            	console.error('invalid expression.');
                return;  
            } 
            var name = matches[1];
            var expression = matches[3];
            
            if(!this) return [];
            var result=[];
            this.forEach(function(value, index, array) {
            	eval('var rpr=/'+ name + '/g');
			   var newexp = expression.replace(rpr,value);
			   var res = eval(newexp);
			   if(res){
			   	 result.push(value);
			   }
			});
			return result;
        };
        Array.prototype.select = function(){
            var args = arguments[0].toString();
            var matches = args.match(/(\w)(\s+)?=>(.*)+/);
            if(!matches){
            	console.error('invalid expression .');
                return;  
            } 
            var name = matches[1];
            var expression = matches[3];
            matches = expression.match(/\.(\w+)/);
            if(!matches)    {
            	console.error('invalid expression .');
                return;  
            } 
            var property = matches[1];
            var result=[];
            this.forEach(function(value, index, array) {
			   if(value[property]){
			   	 result.push(value[property]);
			   }
			});
			return result;
        };
        Array.prototype.distinct = function(){
            var args = arguments[0].toString();
            var matches = args.match(/(\w)(\s+)?=>(.*)+/);
            if(!matches){
            	console.error('invalid expression .');
                return;  
            } 
            var name = matches[1];
            var expression = matches[3];
            matches = expression.match(/\.(\w+)/);
            if(!matches)    {
            	console.error('invalid expression .');
                return;  
            } 
            var property = matches[1];
            var result=[];
            this.forEach(function(value, index, array) {
			   if(value[property]){
			   	 var add = true;
			   	 for (var i = 0; i < result.length; i++) {
			   	 	if(value[property]==result[i][property]){ 
			   	 		add = false; 
			   	 		break;  
			   	 	}
			   	 }
			   	 if(add) result.push(value);
			   }
			});
			return result;
        };
        var mres = [20,33,40,89,55].where(x=>x%5==0);
        var seles = [{age:423},{age:120},{name:'leo',age:80}].select(a=>a.age);
        var distic = [{age:423},{age:120},{age:120},
                     {name:'leo',age:80},{name:'leo',age:80}].distinct(a=>a.age);
        console.log(mres);
```
-------------------
结果

![测试结果](./1.png)

###<center>**性能测试**</center>

  昨天看一个篇不错的[文章](http://www.bonashen.com/post/develop/ria-develop/2015-04-08-javascript-lambda-bian-yi-qi-shi-xian)里面使用了10万数据来测试eval及function，看到这个那我也来试试。代码修改如下：
``` javascript
        var arrs = [],models=[];
        for (var i = 0; i < 100000; i++) {
        	arrs.push(i);
        	models.push({age:i,name:'name'+i});
        }
        
        console.time("where");
        var mres = arrs.where(x=>x%5==0);
        console.timeEnd("where");
        console.time("select");
        var seles = models.select(a=>a.age);
        console.timeEnd("select");
        console.time("distinct");
        var distic = models.distinct(a=>a.age);
        console.timeEnd("distinct");
```
相比distinct，其他两个 很快就能看到结果，过了7-8分钟还是没有结果，后来看了下代码，自己真是笑死。
distinct使用了一个最坏的情况来运行，arrs中没有重复数据，时间复杂度是O(n<sup>2</sup>)，这么看没有个把钟出不来. 那就先让他自己运行吧，刚好和妹子有约，晚上回来在看。晚上11点半到家，结果如下：

![这里写图片描述](./all.png)
结果和预期差不多，一个半小时，chrome按F12在debug模式下会慢些。既然如此那就来调整下，一个是where的实现，一个distinct的是实现，select功能少暂时可以不用考虑。
###**where**
先看使用eval的实现：
![这里写图片描述](./whereeval.png)
然后是function的实现：

![这里写图片描述](./wherefunc.png)
最后看结果：

![这里写图片描述](./whereresult.png)
从结果看这差距不是一般的大。

###**distinct**
这个然我想起了之前看到的位图算法，用一个bit位来记录某项记录是否已存在，我们可以使用的最小数据类型只能是float及int，可以考虑使用[强类型数组](https://www.web-tinker.com/article/20101.html) Uint32Array。这里有个问题是我们事先并不能知道某一个数的大小，有可能该数据所在的位置已经超出bit_arr的界限，所以这个需要做判断。如果超出则增加bit_arr的长度。然后我的实现如下：

```javascript
(function(){
        var bit_arr ;
        var get = function(index,offset){
        	var value = bit_arr[index]>>offset;
        	return value & 0x01;
        };
        var set = function(index,offset){
        	bit_arr[index] = bit_arr[index]+(1<<offset); 
        }
       
        window.distinct = function(arr){
        	var count = parseInt(arr.length/32) + 1;
		    bit_arr = new Uint32Array(count);
		    for (var i = 0; i < count; i++) {
		    	bit_arr[i]=0;
		    }
			var results = [];
	        arr.forEach(function(value,index,array){
	        	var arr_index = parseInt(value/32);
	        	var offset = value%32;
	        	if(!bit_arr[arr_index]) bit_arr[arr_index] = 0;

	        	var bit = get(arr_index,offset);
	            if(bit == 0){
		            set(arr_index,offset);
		            results.push(value);
	            }
	        });
	        return results;
        };
    })();
```
测试代码如下：
```javascript
        var arrs = [],models=[];
        for (var i = 0; i < 20000; i++) {
        	arrs.push(i);
        	// models.push({age:i,name:'name'+i});
        }
        for (var i = 0; i < 20000; i++) {
        	arrs.push(i);
        }
        for (var i = 0; i < 40000; i++) {
        	arrs.push(i*2);
        }
        for (var i = 0; i < 20000; i++) {
        	arrs.push(i*3);
        }
        console.time("distinct");
        distinct(arrs);
        console.timeEnd("distinct");
```
测试结果：

![这里写图片描述](./distinct.png)

结果还明显，我多次测试其时间都在130ms左右。
###<center>**总结**</center>

   其实很多时候只要有一个点子一个想法都可以去尝试，做出来后定会有收获。有关lambda部分还有很多功能可以实现，像前面提到的那位博主有考虑到缓存function，不过他的lambda表达式是使用字符串包含的，感觉不太雅观。CSDN的编辑器太操蛋，竟然没有自动保存草稿的功能，害我重写，哎...真不够专业。还是马克飞象好些。先这样了，后续有想法再补充。

 个人github博客地址参见  http://blog.magicleox.com/

有关eval性能的文章：
http://www.nowamagic.net/librarys/veda/detail/1627

https://www.nczonline.net/blog/2013/06/25/eval-isnt-evil-just-misunderstood/