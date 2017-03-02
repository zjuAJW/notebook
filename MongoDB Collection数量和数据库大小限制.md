#MongoDB Collecion数量限制和数据库大小限制  
 
#
##Collection数量
对于Mongo3.4，默认的存储引擎是**WiredTiger**，对Collecion的数量没有限制。  
而对于Mongo3.2以前的版本，默认存储引擎是**MMAPv1**，官方文档的说法是

	For the MMAPv1 the number of namespaces is limited to the size of the namespace file divided by 628
	
	A 16 megabyte namespace file can support approximately 24,000 namespaces. Each collection and index is a namespace.

Namespece File在官方文档中也有说明

	For the MMAPv1 storage engine, namespace files can be no larger than 2047 megabytes.

	By default namespace files are 16 megabytes. You can configure the size using the nsSize option.

同样的，**WiredTiger**引擎也不受这个限制。 

#  
##数据库大小
在官方文档上找到了关于**MMAPv1**的数据库大小的限制,最大32TB，但是没有找到关于**WiredTiger**的

	The MMAPv1 storage engine limits each database to no more than 16000 data files. 
	This means that a single MMAPv1 database has a maximum size of 32TB. 
	Setting the storage.mmapv1.smallFiles option reduces this limit to 8TB.
#
#
##操作系统的限制
为了以防万一，还是写了一个脚本，在一个db里创建10万个collection，看会不会出问题。果然，不出所料，还是出问题了。  

第一次插入到接近500次的时候就停了，查看发现mongo服务直接被干掉了。很奇怪，又运行了一次，还是这样。如果不删除已建立的collection的话，每次运行能插入的数量越来越少。最后在988个collection后，再插入就报错：

		{
			"ok" : 0,
			"errmsg" : "24: Too many open files",
			"code" : 8,
			"codeName" : "UnknownError"
		}

Too many open files是什么鬼？网上查了一下啊，貌似跟Linux系统本身的限制有关。mongo也提供了[相关的文档](https://docs.mongodb.com/manual/reference/ulimit/)  
大体来说，就是Linux本身对某个进程或者某个用户占用的系统资源进行了限制，当然也就包括对mongo的限制。所以我们需要更改Linux的设置，来让mongo能够使用足够多的系统资源。  
Linux常见的一些限制如下（这是公司测试机的一些情况）：

	$ ulimit -a
	core file size          (blocks, -c) 0
	data seg size           (kbytes, -d) unlimited
	scheduling priority             (-e) 0
	file size               (blocks, -f) unlimited
	pending signals                 (-i) 61992
	max locked memory       (kbytes, -l) 64
	max memory size         (kbytes, -m) unlimited
	open files                      (-n) 1024
	pipe size            (512 bytes, -p) 8
	POSIX message queues     (bytes, -q) 819200
	real-time priority              (-r) 0
	stack size              (kbytes, -s) 8192
	cpu time               (seconds, -t) unlimited
	max user processes              (-u) 61992
	virtual memory          (kbytes, -v) unlimited
	file locks                      (-x) unlimited

mongo的建议配置：

	-f (file size): unlimited
	-t (cpu time): unlimited
	-v (virtual memory): unlimited
	-n (open files): 64000
	-m (memory size): unlimited
	-u (processes/threads): 64000



