#Mongo 	内存使用  
# 
Mongo采用的是内存映射存储引擎， 它会把数据文件映射到内存中，如果是读操作，内存中的数据起到缓存的作用，如果是写操作，内存还可以把随机的写操作转换成顺序的写操作，总之可以大幅度提升性能。MongoDB并不干涉内存管理工作，而是把这些工作留给操作系统的虚拟内存管理器去处理，这样做的好处是简化了MongoDB的工作，但坏处是你没有方法很方便的控制MongoDB占多大内存。   

##常驻内存、映射内存和虚拟内存
具体来说，Mongo在内存使用上有几个概念，主要有**常驻内存****、****映射内存和虚拟内存**。**常驻内存**是Mongo在物理内存中明确占有的部分。Mongo在访问数据时，为其赋予一个虚拟地址，Mongo将该虚拟地址传给操作系统内核，由内核映射为真正的物理内存地址，这样，即使内存将此页面清理出内存了，mongo也可以通过虚拟地址来访问（这不就是操作系统的虚拟内存吗？）。而这部分Mongo赋予了虚拟地址的数据页面就构成了**映射内存**，通常情况下，映射内存包含了Mongo访问过的所有数据，其大小约等于数据集的大小。  

Mongo还会为映射内存中的页面额外维护一份虚拟地址，以供日志系统使用，所以Mongo所使用虚拟内存的大小，约为映射内存（或者说整个数据集）大小的两倍。但是要注意的是，映射内存和虚拟内存跟Mongo真正占用的物理内存毫无关系，二者只是Mongo自身维护的两个映射表罢了。

默认情况下，WiredTiger引擎会使用所有空闲的内存，所以mongo占用内存可能会非常高。官方文档：

>	Starting in 3.4, the WiredTiger internal cache, by default, will use the larger of either:  
>	1.   50% of RAM minus 1 GB, or  
>	2.   256 MB.  
>	Via the filesystem cache, MongoDB automatically uses all free memory that is not used by the WiredTiger cache >
>	or by other processes. Data in the filesystem cache is compressed.

但是在公司测试机上测试时，插入的数据大小应该是大于测试机的内存了，但是最终内存只占了44.3%左右，这是个问题，不知道是不是系统自身对进程的有限制。   

最头疼的是，Mongo自己不释放内存，一直占用着很高，网上目前找到的方法都没法用，只能重启mongo服务，但是这样不解决根本问题，再次对collection进行操作，理论上还是会读取数据进到内存，所以对于数据量很大的单个collection，有什么解决方案吗？