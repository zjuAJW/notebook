#Mongo 单个collection大数据量插入性能测试  
#
由于之前发现不能在一个mongo实例下建立太多的collection，所以只能把数据存在一个collction中咯。所以想测试一下Mongo在单个collection中大数据量下的插入性能。


##实验设置
利用mongoimport向单个collection中导入100万条8个int的数据，重复300次。collection上建立了4个键的符合索引。  
##实验结果
![单个collection大数据量下的插入性能](https://github.com/zjuAJW/MarkdownPhoto/blob/master/mongo.png?raw=true)

![第二次，先把mongo服务关掉](https://github.com/zjuAJW/MarkdownPhoto/blob/master/mongo_huge_data_2.png?raw=true)

图中横坐标是插入次数（1～300），纵坐标是每次插入的用时。开始的时候用时35s左右，后期最高点用时2181s。  
数据导入过程CPU使用率一直高达300%多，数据导入结束后内存占用保持在在40%左右（不过貌似mongo是会一直占内存直到内存占满的？）  
出现的问题是：存在数据丢失现象（按道理应该有300×100万的数据，可是最后count了一下貌似只有296000000左右的数据？具体的数值不记得了，
但是少的还是挺多的），而且时间波动明显，有很多向下的尖刺。

##速度变慢的问题
这个是在意料之中的，按照mongo的内存使用，在用完可用的内存后，其插入速度会有明显的下降，确实在曲线上升的那段是内存用完的时间。  

##数据丢失问题
这个就有点严重了，看了一下mongo的日志，有很多连接错误：

	[conn1343] AssertionException handling request, closing client connection: 6 socket exception [SEND_ERROR] for 127.0.0.1:47164

看mongoimport的输出，确实在某几次会出现连接不了的情况：

	Failed: Closed explicitly

	Failed: error connecting to db server: no reachable servers

会报这两种错误，第一个是在插入过程中连接挂掉了，第二种是在mongoimport连接时就没连上。  
网上找了一下，有[类似的问题](http://blog.csdn.net/u010443481/article/details/50912752)，不过也没写具体原因。只是说：
>mongo是先存储在缓存中然后在存入数据库，但是在存入数据库的过程中有可能会对数据库连接出现问题。
  
StackOverflow上有在导入比较大的数据文件时[无法连接的问题](http://stackoverflow.com/questions/33475505/mongodb-mongoimport-loses-connection-when-importing-big-files)：  
>you can use mongoimport option -j. Try increment if not work with 4. i.e, 4,8,16, depend of the number of core you have in your cpu.

>mongoimport --help

>-j, --numInsertionWorkers= number of insert operations to run concurrently (defaults to 1)  
>
>mongoimport -d mietscraping -c mails -j 4 < mails.json

但是试了一下好像没什么作用。


根据上面的报错，soecket exception， 感觉是连接或者网络上的问题，难道是连接数满了？mongoimport之后没有释放连接？  
但是看了一下，import之后，连接的状态都是TIME_WAIT,所以连接是关闭了的。  

![TIME_WAIT状态](https://github.com/zjuAJW/MarkdownPhoto/blob/master/TIME_WAIT.PNG?raw=true)

但是TIME\_WAIT的连接也是要占用连接数的啊，之前看过，ulimit -n 最大的open files 限制是1024，难道是这里超了？
做了一下实验，发现当我连接数（包括TIME\_WAIT状态的）远超1024的时候，竟然还能建立连接插入数据！！难道我理解错了这个1024的意义？手动捂脸。。。。。。  

>**小tips**:mongo的配置文件里有net.maxIncomingConnections选项来限制mongo的可用连接数，但注意，这个连接数会有一个封顶值，就是系统open files 的80%，如果你设置的值大于这个值的话，是没有用的。

  
另外，丢失数据的情况还是在比较靠后的时候才发生的，但是内存占用还没达到峰值，本来怀疑是mongo的内存用满导致了什么问题，这样看来也不能确定。