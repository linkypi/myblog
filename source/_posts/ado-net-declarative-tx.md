---
title: 将Ado.net中的事务抽离到业务层
date: 2017-06-02
tags:
- Aop
- IL
categories: 
- C# 
- Aop
---

&emsp; &emsp;最近在实现一个简单的审批功能，涉及到一些事务的处理。一个管控台项目，我使用的是最简单的三层架构。使用的是Ado.net操作数据库。先看一段我们经常用到的事务代码：
```cs
SqlConnection con = new Sqlconnection("数据库连接语句");
con.Open();
var trans = con.BeginTransaction();
try
{
     SqlCommand com = new SqlCommand(trans);
     //处理插入或更新逻辑
     trans.Commit();
}catch(ex){
     trans.Rollback();//如果前面有异常则事务回滚
}
finally
{
     con.Close();
}
```
 &emsp; &emsp;我一直以来都是使用下面这种方式在Dao来处理事务，其实怎么看都觉得他别扭，就拿我做的审批功能来说，当前审批人通过之后需要生成一条审批记录（记作表ApprovalHistory），同时将当前申请单的当前审批人指向下一个处理者(记作Apply)，而一般的三层架构都会有自身的Service层，理想的情况应该是在Service层用事务来处理相应的逻辑。
       正常的处理逻辑应该是将事务提取出来，应用层不应该过多的去关心底层的实现。乱糟糟的代码写在一起分层感觉也没啥意思了。如果可以像Spring那样使用Annotation的方式就一行代码来实现事务这样不是很好吗。这个问题确实想了挺久，后来有位boss和我说了下他的实现，喜出望外！先贴贴代码实现
#### **Boss的实现**

```cs
    public class ConnId
    {
        private int mConnId = 0;
        private DateTime mCreateTime = DateTime.Now;
                 
        public int MConnId
        {
            get { return this.mConnId; }
        }

        public ConnId(int connid)
        {
            this.mConnId = connid;
        }
    }
	
    public class DbConnection
	{
        private static readonly string sConnStr = "从配置中读取连接字符串";
        public static readonly long MAX_LAST_TIME_LEN = 10 * 1000 * 1000 * 60; 

        private ConnId mConnId = null;
        private SqlConnection mConn = null;
        private SqlCommand mSqlCmd= null;
        private DateTime mCreateTime = DateTime.Now;
        private string mTransName = "";

        public ConnId MConnId
        {
            get { return this.mConnId; }
        }

        public SqlConnection Connection
        {
            get { return this.mConn; }
        }

        public SqlCommand SqlCmd
        {
            get { return this.mSqlCmd; }
        }

        public ConnectionState State
        {
            get
            {
                if (mConn == null)
                {
                    return ConnectionState.Broken;
                }
                else
                {
                    return mConn.State;
                }
            }
        }

        public DbConnection()
        {
        }

        internal ConnId ConnOpen(HttpRequest request)
        {
            try
            {
                this.mConn = new SqlConnection(sConnStr);
                this.mSqlCmd = new SqlCommand();
                mSqlCmd.Connection = this.mConn;
                this.mConnId = new ConnId(this.GetHashCode());
                mConn.Open();
                if (request == null)
                {
                    mTransName = "null";
                }
                else
                {
                    mTransName = GetSrcFileName(request);
                }
                mSqlCmd.Transaction =
                    mConn.BeginTransaction(System.Data.IsolationLevel.ReadCommitted,
                    mTransName);
                //log

            }
            catch (Exception e)
            {
                //log
                if (this.mConn.State != System.Data.ConnectionState.Closed)
                {
                    this.mConn.Close();
                    this.mConn.Dispose();
                }
                this.mConnId = null;
            }
            return this.mConnId;
        }

        internal ConnId ConnOpen(string src)
        {
            try
            {
                this.mConn = new SqlConnection(sConnStr);
                this.mSqlCmd = new SqlCommand();
                mSqlCmd.Connection = this.mConn;
                this.mConnId = new ConnId(this.GetHashCode());
                mConn.Open();
                mTransName = src;
                mSqlCmd.Transaction =
                    mConn.BeginTransaction(System.Data.IsolationLevel.ReadCommitted,
                    mTransName);
                //log
            }
            catch (Exception e)
            {
                //log
                if (this.mConn.State != System.Data.ConnectionState.Closed)
                {
                    this.mConn.Close();
                    this.mConn.Dispose();
                }
                this.mConnId = null;
            }
            return this.mConnId;
        }

        internal Exception Cancel()
        {
            Exception ex = null;
            try
            {
                //log
                mSqlCmd.Transaction.Rollback();
            }
            catch (Exception e)
            {
                //log
                mSqlCmd.Transaction.Rollback();
                ex = e;
            }
            finally
            {
                if (mSqlCmd != null)
                {
                    this.mSqlCmd.Dispose();
                }
                if (this.mConn.State != System.Data.ConnectionState.Closed)
                {
                    this.mConn.Close();
                }

                memberClear();
            }
            return ex;
        }

        internal Exception Close()
        {
            Exception ex = null;
            try
            {
                //log
                mSqlCmd.Transaction.Commit();
            }
            catch (Exception e)
            {
                //log
                mSqlCmd.Transaction.Rollback();
                ex = e;
            }
            finally
            {
                if (mSqlCmd != null)
                {
                    this.mSqlCmd.Dispose();
                }
                if (this.mConn.State != System.Data.ConnectionState.Closed)
                {
                    this.mConn.Close();
                    this.mConn.Dispose();
                }

                memberClear();
            }
            return ex;
        }

        internal bool IfExpried()
        {
            if (mConnId != null)
            {
                if (mCreateTime != DateTime.MinValue)
                {
                    DateTime now = DateTime.Now;
                    if (now.Ticks - this.mCreateTime.Ticks > MAX_LAST_TIME_LEN)
                    {
                        return true;
                    }
                }
            }
            return false;
        }

        private void memberClear()
        {
            mConnId = null;
            mConn = null;
            mSqlCmd= null;
            mCreateTime = DateTime.MinValue;
        }

        private string GetSrcFileName(HttpRequest r)
        {
            FileInfo fi = new FileInfo(r.PhysicalPath);
            string filename = fi.Name.Replace(fi.Extension, "");
            if (filename.Length > 32)
            {
                filename = filename.Substring(filename.Length - 32, 32);
            }
            return filename;
        }

        private string GetSrcFileName(string src)
        {
            return src.Substring(src.Length - 32, 32);
        }

        private string GetShortTime(DateTime t)
        {
            string str = t.Day.ToString() + "_" + t.Hour.ToString() + ":" +
                t.Minute.ToString() + ":" + t.Second.ToString();
            return str;
        }
	}

    public class ConnPool
	{
        private static readonly object sLocker = new object();
        private static Dictionary<ConnId, DbConnection> sConnList = new Dictionary<ConnId, DbConnection>(MAX_CONCURRENT_CONN_COUNT);
        private static readonly int MAX_CONCURRENT_CONN_COUNT = 1000;

        public static int sCount
        {
            get { return sConnList.Keys.Count; }
        }

        public static ConnId CreateConn(HttpRequest request)
        {
            DbConnection dbconn = null;
            ConnId key = null;
            try
            {
                dbconn = new DbConnection();
                key = dbconn.ConnOpen(request);
                if (sConnList.ContainsKey(key))
                {
                    return key;
                }
                if (sCount < MAX_CONCURRENT_CONN_COUNT)
                {
                    lock (sLocker)
                    {
                        if (sCount < MAX_CONCURRENT_CONN_COUNT)
                        {
                            sConnList.Add(key, dbconn);
                        }
                        else
                        {
                           //log
                        }
                    }
                }
            }
            catch (Exception e)
            {
                //log
                key = null;
            }
            return key;
        }

        public static ConnId CreateConn(string src)
        {
            DbConnection dbconn = null;
            ConnId key = null;
            try
            {
                dbconn = new DbConnection();
                key = dbconn.ConnOpen(src);
                if (sCount < MAX_CONCURRENT_CONN_COUNT)
                {
                    lock (sLocker)
                    {
                        if (sCount < MAX_CONCURRENT_CONN_COUNT)
                        {
                            sConnList.Add(key, dbconn);
                        }
                        else
                        {
                            //log
                        }
                    }
                }
            }
            catch (Exception e)
            {
                //log
                key = null;
            }
            return key;
        }

        public static int ReleaseConn(ConnId connid)
        {
            DbConnection conn = null;
            try
            {
                lock (sLocker)
                {
                    conn = sConnList[connid];
                    sConnList.Remove(connid);
                }
                if (conn != null)
                {
                    conn.Close();
                }
            }
            catch (Exception e)
            {
                //log
            }
            return 0;
        }

        public static int CancelConn(ConnId connid)
        {
            DbConnection conn = null;
            try
            {
                lock (sLocker)
                {
                    conn = sConnList[connid];
                    sConnList.Remove(connid);
                }
                if (conn != null)
                {
                    conn.Cancel();
                }
            }
            catch (Exception e)
            {
                //log
            }
            return 0;
        }

        public static DbConnection GetDbConn(ConnId connid)
        {
            return sConnList[connid];
        }
	}
```
  然后使用的时候呢就像这样就可以：
```cs
    namespace MyTest
    {
	    class Program
	    {
	        static void Main(string[] args)
	        {
	            ConnId conn = ConnPool.CreateConn("123");
	            try
	            {
	                var aservice = new ApplyService();
	                var historyService = new ApprovalHistoryService();
	                aservice.Update(new object(), conn);
	                historyService.Insert(new object(), conn);
	
	                ConnPool.ReleaseConn(conn);
	            }
	            catch (Exception ex)
	            {
	                ConnPool.CancelConn(conn);
	            }
	        }
	    }
    }
```
 &emsp; &emsp;可以看到这里主要是在ConnPool中使用一个Dictionary加双重检查锁定来实现一个并发连接的处理，用于记录当前的数据库连接。而到执行Insert和Update时他就通过connid在ConnPool中取出对应事务中的SqlCommand来执行相应的Sql 。也就是说他将数据库连接及事务抽离，放到了一个ConnPool中管理。基本符合我预期。
#### **问题来了**
  &emsp; &emsp;如果认真看着里面的实现大家应该会发现这里存在一些问题。
###### **1 并发字典的实现**
   &emsp; &emsp;头一个我想到的就是双重检查锁定的问题，虽然老总说他们用了那么久一直没有什么问题，但我想说那是因为并发量不大所以没有发现问题，并发量大的情况下使用lock的性能是明显下降的，这就让我想起了Java中从HashMap 到 HashTable 再到 ConcurrentHashMap的转变。HashMap是非线程安全类，所以用在多线程环境下就很可能出现意想不到的结果。所以才有了HashTable，我印象当中HashTale的实现是在HashMap的基础上为每个方法加了synchronized关键字，所以每次Add或Remove都会锁住整个内部的数组，可以想象一下在多线程环境下这里面的操作会有多慢。所以才有了ConcurrentHashMap的实现，其内部使用的是可重入锁，而锁住的是每一个segment段而不是整个数组。更重要的是锁的实现（基础框架是队列同步器AbstractQueuedSynchronizer），追究到最底层实现是使用CAS加自旋，一种乐观锁的方式来实现，从而保证了并发性。从HashMap 到 HashTable 再到 ConcurrentHashMap的转变真让我回味良久，从里面数据结构的设计到并发的处理真是妙不可言。学习Java的朋友应该知道这可以从放腾飞中的《并发编程的艺术》中了解到，初学者看可能会晕，我第一次看了一小部分后是拒绝的，因为看得想吐，心里在咒骂这他妈都写的什么鬼，哈哈！再后来慢慢看就有所体会了，而且有些地方还要反复琢磨。从这本书可以了解到很多并发编程的底层实现，极力推荐！！！
         &emsp;&emsp;所以我也并不推荐自己去实现一个线程安全的Dictionary，因为里边涉及到太多的细节不是我们所能预料的，除非自己真的非常熟悉底层的实现，对并发编程了然于胸。看过.Net中的 Dictionary实现后会发现它与HashMap的实现大体无异，虽然没有看过.Net中ConcurrentDictionary的实现，但是个人感觉他们的实现大体上应该相差无几。所以可以考虑使用ConcurrentDictionary来替换老总ConnPool的内部实现，这部分代码就不贴了。
###### **2 非托管资源的释放**
 &emsp; &emsp;对于非托管资源的释放我建议是使用继承接口IDisposable来实现其Dispose()方法，具体请参考.Net圣经 -《CLR via C# 第4版 》。
  &emsp; &emsp;
####**改进**
&emsp;&emsp; 虽然老总的实现能抽离的底层的实现，但是还不够优雅，因为在代码的最后都要去手动实现事务的提交和回滚。那有没有更好的办法来实现不用手动提交和回滚呢，就像Spring中事务，只需要在方法中加注解就可以达到指定功能。当然初期先来一个简单的实现。.Net中的Attribute对应的就是Java中的注解，但是这个Attribute还必须具备Aop的功能。这样才可以在方法执行前开启一个事务，方法执行完成后提交或回滚事务。
##### **1. Aop**
&emsp;&emsp; 一般使用的较多的是Autofac和Castle，当然还可以使用Remoting代理方式或者从ContextBoundObject中派生来实现。刚好找到一篇文章说到了[.net实现Aop的七种方式](https://ayende.com/blog/2615/7-approaches-for-aop-in-net)。

| Approach                                       |                                                Advantages |                        Disadvantages                         |
| :--------------------------------------------- | --------------------------------------------------------: | :----------------------------------------------------------: |
| Remoting Proxies                               |  Easy to implement, because of the .Net framework support | Somewhat heavyweight  Can only be used on interfaces or MarshalByRefObjects |
| Derivingfrom ContextBoundObject                | Easiest to implement Native support for call interception |             Very costly in terms of performance              |
| Compile-time subclassing ( Rhino Proxy )       |                                     Easiest to understand |              Interfaces or virtual methods only              |
| Runtime subclassing ( Castle Dynamic Proxy )   |                       Easiest to understand Very flexible | Complex implementation (but alreadyexists) Interfaces or virtual methods only |
| Hooking into the profiler API ( Type Mock )    |                                        Extremely powerful | Performance? Complex implementation (COM API, require separate runner, etc) |
| Compile time IL-weaving ( Post Sharp / Cecil ) |                            Very powerful Good performance |                    Very hard to implement                    |
| Runtime IL-weaving ( Post Sharp / Cecil )      |                            Very powerful Good performance |                    Very hard to implement                    |
当然这只是前人做的一个总结，具体的性能及优缺点还需要自己去考量。
编译时AOP工具有：PostSharp、LinFu、SheepAspect、Fody、CIL操作工具。
运行时AOP工具：Castle Windsor/DynamicProxy、StructureMap、Unity、Spring.NET。
&emsp;&emsp; 我用得比较多的是运行时Aop，比如Castle、Autofac.他们都是使用动态代理的方式来实现。来看看Castle是怎么实现的
```cs
     using Castle.DynamicProxy;
     using System;
     
    class Program
    {
        static void Main(string[] args)
        {
            ProxyGenerator generator = new ProxyGenerator();
            SimpleInterceptor interceptor = new SimpleInterceptor();
 
            Person person = generator.CreateClassProxy<Person>(interceptor);
            person.SayHello();

            Console.Read();
        }
    }
 
    public class Person
    {
        public virtual void SayHello()
        {
            Console.WriteLine("hello world.");
        }

        public virtual void SayName(string hometown)
        {
            Console.WriteLine("I'm Lynch, I'm from {0}.", hometown);
        }

        public void SayOther()
        {
            Console.WriteLine("Yeah, I'm Chinese.");
        }
    }
     
    public class SimpleInterceptor : StandardInterceptor
    {
        protected override void PreProceed(IInvocation invocation)
        {
            Console.WriteLine("before invocation , method : {0}.", invocation.Method.Name);
            base.PreProceed(invocation);
        }

        protected override void PerformProceed(IInvocation invocation)
        {
            Console.WriteLine("before performing ...");
            base.PerformProceed(invocation);
            Console.WriteLine("after performing...");
        }

        protected override void PostProceed(IInvocation invocation)
        {
            Console.WriteLine("after invacation , method : {0}.", invocation.Method.Name);
            base.PostProceed(invocation);
        }
    } 
```
动态代理的方式有个不好的地方就是每次都要生成指定类型的代理类，要实现Aop的方法还必须是virtual方法. 如果有很多类很多方法需要拦截那增加和改动的代码就有点多。我想达到的目标是只要一个Attribute类，不需要生成指定类型的代理类，让代码看起来更加的干净。一直以来我只知道有运行时Aop，就没有想到编译时Aop，比如postsharp。然后就找到了Mono.Cecil ，通过改写中间IL代码的方式来实现，大体思路是

1. 找到标记有指定AopAttribute的方法
2. 复制该方法并生成一个新的方法copy_method，复制完成后清楚原有方法
3. 改写原有方法，首先调用AopAttribute的OnStart方法，接着调用copy_method ，调用AopAttribute的OnSuccess方法，最后调用AopAttribute的OnEnd方法

> postsharp 1.5 使用注意事项，.net Framework 必须是3.5版本，需要在csproj中增加以下内容
```
  <PropertyGroup>
    <PostSharpUseCommandLine>True</PostSharpUseCommandLine>
    <DontImportPostSharp>True</DontImportPostSharp>
    <PostSharpDirectory>libs</PostSharpDirectory>
  </PropertyGroup>
  
 <Import Project="$(PostSharpDirectory)\PostSharp-1.5.targets" />
```
#####  **2. IL **
&emsp;&emsp;说到IL指令就要先知道什么是evaluation stack。而这个evaluation stack却不同于我们平时理解的Call Stack（调用栈），即在调用一个方法时首先会将所有参数压栈，压栈完成后调用指定方法，方法执行完成清理刚刚入栈的参数。我写这个IL指令的时候我也纳闷了，我就在想我大学的时候用Intel x86汇编自己实现小型操作系统的时候也没有遇到这个东西啊，说到栈就Call Stack，这evaluation stack（以下简称EStack）他妈是什么鬼。后来向朋友了解了一下才知道这个是CIL特有的东西，这个是个寄存器，即`Stack< Register >` 。这样就说通了，我忘了操作结果的存放，学过汇编或了解一些底层的同学应该了解，汇编语言的操作结果都是存放的寄存器中，如32位的ax , bx，64位 eax、ebx等通用寄存器。而不同的CPU又有不同的指令集，如PC机使用的是x86复杂指令集，而Arm使用的是Arm的精简指令集，而CLR直接将兼容不同的寄存器的工作交给使用者处理的话那使用者势必想疯掉，所以VM做一个通用的寄存器来存放操作结果，至于该使用哪个寄存器来存放使用者不需要关心。
 &emsp;&emsp;至于为什么设计成栈的结构，个人理解一个是栈有内存限制，我们一般使用到的临时变量局部变量或者方法参数都不会太多，当然也可能很多，太多参数的话就该考虑封装了。二个我觉得更重要的是它非常符合调用方法前将参数出入Call Stack的操作，例如我们来执行一下伪代码：

```
            int a = 123;
            Service service = new Service("lynch");
            var b = service.GetNumber();
            var result = service.Add(a,b);
```
在执行Add方法前会先调用GetNumber来获取b的值，整个代码的执行指令是

```
IL_0001:  ldc.i4.s  123   将123赋值给寄存器即放到EStack中 
IL_0003:  stloc.0         将EStack中索引为0的值出栈，并将出栈的值push到Call Stack作为Add方法的入参
```
限于篇幅剩下的IL代码就不贴了，从第二行的st前缀指令大家应该可以发现 : 这个指令包含了两个操作，一个从EStack出栈，二个将出栈的值入栈Call Stack。EStack是通过ld入栈而st出栈，就是说使用到某个参数的时候就将其从EStack出栈，而无需再占用栈空间，也就释放了栈内存，是不是有点像Call Stack的操作。个人一些见解，不足之处还望指正。

&emsp;&emsp;在IL指令中我们会频繁用到如ld ( load )、st(store)等前缀指令，ld前缀指令的意思就是将寄存器的值压栈，也就是将EStack中的值push到Call Stack，而st前缀指令就是将Call Stack中的操作结果存放到寄存器EStack中。大家可以通过这篇文章做个基本了解
- [Understanding Common Intermediate Language (CIL)](https://www.codeproject.com/Articles/362076/Understanding-Common-Intermediate-Language-CIL)

如果想深入了解的可以看《Inside Microsoft .NET IL Assembler》，中文版是《Microsoft.NET IL汇编语言程序设计》，不过中文版已经绝版，网上可以找到很多影印版pdf。
##### **3. Aop的实现**

&emsp;&emsp;这个应该算是postsharp的简单实现，源码放在了Github [LeoxAop](https://github.com/linkypi/LexoAop)上。代码我就不贴了，很多地方都有注释，而且逻辑还算清晰。这里的实现部分参考了[MSBuild + MSILInect实现编译时AOP之预览](http://www.cnblogs.com/whitewolf/archive/2011/08/09/2132217.html)这位博主的实现，不过他写的应该过于匆忙，所以代码结构有些凌乱，不太容易看懂，还用了很多的linq。 
&emsp;&emsp;这里有一个待解决的问题是将代码注入到指定项目exe或dll后怎么让VS调试到指定的AopAttribute代码，也就是说怎么生成对应的pdb文件让VS感知到。就像.net reflector 一样，反编译dll后自动生成对应的pdb文件，然后就可以顺利的调试。我目前想到的较为简单的方法是在开发者命令中使用ildasm 将文件反编译为il代码，然后再使用ilasm生成对应的pdb文件 : 
```cmd
ildasm test.exe /out=test.il
ilasm test.il /pdb
```
不过我试过发现并没有起作用，哪位朋友知道的麻烦告诉我一下，万分感激。就因为这个也花了不少时间，搜google搜codeproject都没有找到相关的文章，实在没办法先搁放在这里，太过纠结容易崩溃。本以为很快能结篇，还不料涉及的东西有点多，写代码调试解决遇到的bug也花不少时间。这里是.net 的实现，其实java也可以这么实现，只是要了解java的字节码，有时间再写吧。

扩展阅读
- [栈式虚拟机和寄存器式虚拟机？](https://www.zhihu.com/question/35777031)
##### **4. Transaction 实现**
&emsp;&emsp;终于写到了这了，迫不及待啊。感叹时间飞快。写完这个接下来我想看的东西就是node里边Promise和Async的实现。既然Aop功能已经实现，那我们就可以在OnStart方法开始一个事务，在OnSuccess和OnException提交或回滚事务，但是这里还有几个问题需要考虑：
1. 就是如果使用者的在自身业务代码就已经做了异常捕获该如何处理，是该回滚还是该提交，这个还没想出来好的解决办法。
2. 如果有部分连接未能及时释放又该如何处理，对于这个问题可以考虑启动一个线程来监控，根据连接开始创建的时间来做判断。
3. 底层dao操作需要用到当前连接创建的SqlCommand，要获取到这个那就需要记录连接Id，问题是这个Id只有在相应Aop的On事件时才能拿到。还有没有其他办法呢，因为每个线程执行时用的连接Id都不一样所以我想到一个办法就是将这个变量放到ThreadLocal中，线程跑到哪里它就跟到哪，每个线程都维持着自己的连接Id。如果这个问题不解决那设计这一整个抽象事务就没用了。可能应该还有更好的办法，还没来得及去看Spring中实现，如果有朋友想到更好的办法麻烦告诉我一声，感激不尽！
4. 连接Id如何保证唯一性。当前我使用的是guid生成，只是一个临时的策略。考虑到分布式架构的话这种生成方式就不太好管理，也不稳妥。我比较喜欢 Twitter 分布式自增Id的实现snowflake，说喜欢是因为它的实现粒度很细，但是没有考虑到它强依赖机器时钟，如果机器上时钟回拨，会导致发号重复或者服务会处于不可用状态。当然也可以参考MongoDb中ObjectId 的生成方式。后来有幸看到朋友转发的一篇文章，里面有说到唯一Id的多种生成方式，还介绍了snowflake及ObjectId的优缺点，最后讲到一种新的生成算法Leaf，大家可以了解下
  [Leaf——来自美团点评的分布式ID生成系统](http://mp.weixin.qq.com/s/mY6ulc62pP77Iyl3CiK3dw)
  &emsp;&emsp;当然如果是分布式架构那没这么简单了，需要考虑分布式事务，涉及两阶段三阶段提交、分布式一致性算法 paxos。不过现在更多的应该考虑放弃强一致性的分布式事务而使用最终一致性。如eBay的实现，在设计上就不采用分布式事务，而是通过其它途径来解决数据一致性问题。其中使用的最重要的技术就是消息队列和消息应用状态表。至于阿里和京东怎么实现就没有深入了解过。eBay 实现参考 :
-  [Base: An Acid Alternative](http://queue.acm.org/detail.cfm?id=1394128)

最后 Transaction 的实现请参见 [Leox.Transaction](https://github.com/linkypi/Leox.Transaction)