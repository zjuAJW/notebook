#Mongo 单个collection大数据量插入性能测试  
#
由于之前发现不能在一个mongo实例下建立太多的collection，所以只能把数据存在一个collction中咯。所以想测试一下Mongo在单个collection中大数据量下的插入性能。


##实验设置
利用mongoimport向单个collection中导入100万条8个int的数据，重复300次。collection上建立了4个键的符合索引。  
##实验结果
![单个collection大数据量下的插入性能](https://github.com/zjuAJW/MarkdownPhoto/blob/master/mongo.png?raw=true)

图中横坐标是插入次数（1～300），纵坐标是每次插入的用时。开始的时候用时35s左右，后期最高点用时2181s。  
数据导入过程CPU使用率一直高达300%多，数据导入结束后内存占用保持在在40%左右（不过貌似mongo是会一直占内存直到内存占满的？）  
出现的问题是：存在数据丢失现象（按道理应该有300×100万的数据，可是最后count了一下貌似只有296000000左右的数据？具体的数值不记得了，但是少的还是挺多的），而且时间波动明显，有很多向下的尖刺。
