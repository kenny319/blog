# 从Caelus优化深入JIT
Caelus是唯品会**自主研发**的**高性能**的**分布式**数据库连接池，主要特性：

* 支持分布式数据库：配置管理多个数据库的连接
* 支持同实例不同数据库内连接复用：解决目前很多业务系统在存在一个数据库实例多个数据库的情况下，使用传统连接池时不同数据库的连接池之间无法共享连接导致数据库连接数过多的问题。
* 更低的资源消耗：在大量分库的情况下，使用传统连接池导致的线程数过多的问题
* 更高的性能：通过连接复用、降低资源消耗、事务指令优化、连接池JIT友好性优化等手段提高性能

## Caelus性能
先来了解一下Caelus具体的性能，以下是一组测试数据：
![Alt Image Text](caelus.jpg "Caelus Performance")
连接池最主要的就是获取连接和关闭连接的性能。这是一个通过不同数量的并发线程执行500W次从连接池获取连接并释放连接的测试用例，从上图可知C3P0所用的总时间是Caelus的300倍，Druid所用的总时间是Caelus的12倍左右。

那么为什么Caelus可以做到如此优异的性能呢，关键在于Caelus基于一个比较优秀的开源项目HikariCP做的实现。HikariCP在很多地方都做了优化：  

	* 获取连接的策略优化
	* 无锁优化
	* 数据结构优化
	* JIT友好性优化

本文重点将展示HikariCP在实现代码JIT友好性上所做的努力，同时延伸介绍JIT以及Inlining(方法内联）.

## HikariCP的JIT优化
为了达到更高的性能，HikarciCP在JIT优化方面做了很多努力，甚至部分介绍JIT的文章也会将它作为一个案例。例如JITWatch的作者在介绍JITWatch的时候也提到了HikariCP: [Why it rocks to finally understand Java JIT with JITWatch](http://zeroturnaround.com/rebellabs/why-it-rocks-to-finally-understand-java-jit-with-jitwatch/) (可能需要翻墙），接下来通过例子一起来窥探一下。

跟大部分其他连接池一样，代理JDBC相关的接口来处理SQL执行过程中的异常是常规的手段，HikariCP同样的代理了几乎所有的JDBC的接口，增加了异常处理的代码。为了让HikariCP跑的更快，通过研究方法的字节码输出来优化方法使得方法的字节码大小不超过JIT方法内联的阈值即35个字节。以下就是这个异常处理的方法的优化过程：

### Example one

```java
public SQLException checkException(SQLException sqle) {
        String sqlState = sqle.getSQLState();
        if (sqlState == null)
            return sqle;

        if (sqlState.startsWith("08"))
            _forceClose = true;
        else if (SQL_ERRORS.contains(sqlState))
            _forceClose = true;
        return sqle;
}
```
通过运行如下JDK命令查看生成的字节码：

```shell
javap -public -l -v com.vip.venus.caelus.pool.ProxyConnection    
```
生成的字节码超过了35：

```java
public java.sql.SQLException checkException(java.sql.SQLException);
    descriptor: (Ljava/sql/SQLException;)Ljava/sql/SQLException;
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=2
         0: aload_1
         1: invokevirtual #168                // Method java/sql/SQLException.getSQLState:()Ljava/lang/String;
         4: astore_2
         5: aload_2
         6: ifnonnull     11
         9: aload_1
        10: areturn
        11: aload_2
        12: ldc           #173                // String 08
        14: invokevirtual #175                // Method java/lang/String.startsWith:(Ljava/lang/String;)Z
        17: ifeq          28
        20: aload_0
        21: iconst_1
        22: putfield      #181                // Field _forceClose:Z
        25: goto          45
        28: getstatic     #69                 // Field SQL_ERRORS:Ljava/util/Set;
        31: aload_2
        32: invokeinterface #183,  2          // InterfaceMethod java/util/Set.contains:(Ljava/lang/Object;)Z
        37: ifeq          45
        40: aload_0
        41: iconst_1
        42: putfield      #181                // Field _forceClose:Z
        45: aload_1
        46: areturn
```

### 第一次优化

```java
String sqlState = sqle.getSQLState();
if (sqlState != null && (sqlState.startsWith("08") || SQL_ERRORS.contains(sqlState)))
        _forceClose = true;
return sqle;
```
优化后字节码下降到了36。

### 再一次优化

```java
String sqlState = sqle.getSQLState();
_forceClose |= (sqlState != null && (sqlState.startsWith("08") ||SQL_ERRORS.contains(sqlState)));
return sale;
```
结果情况更加糟糕了，又变成了45。

### 最后的结果

```java
String sqlState = sqle.getSQLState();
if (sqlState != null)
    _forceClose |= sqlState.startsWith("08") | SQL_ERRORS.contains(sqlState);
return sqle;
```
再次查看字节码，惊喜的发现字节码降到了35个字节。

```java
0: aload_1
1: invokevirtual #153                // Method java/sql/SQLException.getSQLState:()Ljava/lang/String;
 4: astore_2
 5: aload_2
 6: ifnull        34
 9: aload_0
10: dup
11: getfield      #149                // Field forceClose:Z
14: aload_2
15: ldc           #157                // String 08
17: invokevirtual #159                // Method java/lang/String.startsWith:(Ljava/lang/String;)Z
20: getstatic     #37                 // Field SQL_ERRORS:Ljava/util/Set;
23: aload_2
24: invokeinterface #165,  2          // InterfaceMethod java/util/Set.contains:(Ljava/lang/Object;)Z
29: ior
30: ior
31: putfield      #149                // Field forceClose:Z
34: return
```

字节码越小的方法不仅更有机会被JIT内联，同时也更能节约code cache的空间使得更多的代码拥有机会被JIT编译，另外CPU Cache也能容纳更多的代码。

那么到底什么是JIT，又有什么方法可以知道自己写的代码是否触发了JIT? 什么是方法内联，哪些方法被JIT进行内联优化了呢？触发内联的条件又是什么呢？

### Example two
为了给PrepareStatement创建代理对象，一开始实现了一个单例的工厂类，PROXY_FACTORY是该单例的一个对象作为ConnectionProxy的一个静态属性。

```java
public final PreparedStatement prepareStatement(String sql, String[] columnNames) throws SQLException
{
    return PROXY_FACTORY.getProxyPreparedStatement(this, delegate.prepareStatement(sql, columnNames));
}
```
该方法生成的字节码如下：

```java
public final java.sql.PreparedStatement prepareStatement(java.lang.String, java.lang.String[]) throws java.sql.SQLException;
    flags: ACC_PRIVATE, ACC_FINAL
    Code:
      stack=5, locals=3, args_size=3          //操作数栈的大小为5,本地变量表为3，参数数量为3(this为实例方法的隐参）
         0: getstatic     #59                 // Field PROXY_FACTORY:Lcom/zaxxer/hikari/proxy/ProxyFactory;获取静态变量并入操作数栈
         3: aload_0							      // 从本地变量表加载this并入操作数栈
         4: aload_0                           // 从本地变量表加载this并入操作数栈
         5: getfield      #3                  // Field delegate:Ljava/sql/Connection;
         8: aload_1							      //从本地变量表加载第一个参数并入操作数栈
         9: aload_2							      //从本地变量表加载第二个参数并入操作数栈
        10: invokeinterface #74,  3           // InterfaceMethod java/sql/Connection.prepareStatement:(Ljava/lang/String;[Ljava/lang/String;)Ljava/sql/PreparedStatement;
        15: invokevirtual #69                 // Method com/zaxxer/hikari/proxy/ProxyFactory.getProxyPreparedStatement:(Lcom/zaxxer/hikari/proxy/ConnectionProxy;Ljava/sql/PreparedStatement;)Ljava/sql/PreparedStatement;
        18: areturn
```
尝试使用静态方法代替单例优化后的方法如下：

```java
public final PreparedStatement prepareStatement(String sql, String[] columnNames) throws SQLException
    {
        return ProxyFactory.getProxyPreparedStatement(this, delegate.prepareStatement(sql, columnNames));
    }
```
优化后的字节码如下：

```java
public final java.sql.PreparedStatement prepareStatement(java.lang.String, java.lang.String[]) throws java.sql.SQLException;
    flags: ACC_PRIVATE, ACC_FINAL
    Code:
      stack=4, locals=3, args_size=3
         0: aload_0
         1: aload_0
         2: getfield      #3                  // Field delegate:Ljava/sql/Connection;
         5: aload_1
         6: aload_2
         7: invokeinterface #72,  3           // InterfaceMethod java/sql/Connection.prepareStatement:(Ljava/lang/String;[Ljava/lang/String;)Ljava/sql/PreparedStatement;
        12: invokestatic  #67                 // Method com/zaxxer/hikari/proxy/ProxyFactory.getProxyPreparedStatement:(Lcom/zaxxer/hikari/proxy/ConnectionProxy;Ljava/sql/PreparedStatement;)Ljava/sql/PreparedStatement;
        15: areturn
```
优化后我们发现字节码变小了，节约了一个getstatic的操作，以及该单例的入栈出栈的操作，同时操作数栈的大小也由5变成了4。另外一个优化点是用invokestatic代替了invokevirtual。我们都知道面向对象的一个核心是多态，虚拟方法的调用可以在运行时决定实际的对象，但是也正是由于这个不确定性导致了对JIT优化不友好，由于无法确定方法所属的实例所以类似于内联等重要的JIT优化无法进行。改成静态方法后，由于明确了方法所以JIT可以对其进行优化。
## JIT

## Inling