
[TOC]

# Zookeeper简介
## 1 概念介绍
> Zookeeper是一个分布式协调服务；就是为用户的分布式应用程序提供协调服务

1. zookeeper是为别的分布式程序服务的
2. Zookeeper本身就是一个分布式程序（只要有半数以上节点存活，zk就能正常服务）
3. Zookeeper所提供的服务涵盖：主从协调、服务器节点动态上下线、统一配置管理、分布式共享锁、统一名称服务……
4. 虽然说可以提供各种服务，但是zookeeper在底层其实只提供了两个功能：
	* 管理(存储，读取)用户程序提交的数据；
	* 并为用户程序提供数据节点监听服务；

## 2 常用应用场景

> 主从协调

![](https://note.youdao.com/yws/api/personal/file/6B338A7DBD924D8699FAF8FE03ADD6B8?method=download&shareKey=6b71f0527afd51f375fa708d27408fd4)

> 配置管理


![](https://note.youdao.com/yws/api/personal/file/98EB0808F91D4895A2A920BEE2045B2D?method=download&shareKey=04688408b12c03ca15ab0ca05ed712b8)

## 3 Zookeeper 集群部署

### 1、Zookeeper集群角色
> Zookeeper集群的角色：  Leader 和  follower  （Observer）
> zk集群最好配成奇数个节点
> 只要集群中有半数以上节点存活，集群就能提供服务

### 2 Zookeeper部署
 
#### 2.1 机器准备

> 1/ 安装到3台虚拟机上
> 2/ 安装好JDK
> 3/ 上传安装包。上传用工具。
> 4/ 解压
~~~
    su - hadoop（切换到hadoop用户）
    tar -zxvf zookeeper-3.4.5.tar.gz（解压）
~~~
> 5/ 重命名
> mv zookeeper-3.4.5 zookeeper（重命名文件夹zookeeper-3.4.5为zookeeper）
> 可以删除里面一些源码工程相关的文件，剩下的是这些：
 
#### 2.2修改环境变量

> （注意：3台zookeeper都需要修改）

> 1/ su – root(切换用户到root)
> 2/ vi /etc/profile(修改文件)
> 3/ 添加内容：

~~~
export ZOOKEEPER_HOME=/home/hadoop/zookeeper
export PATH=$PATH:$ZOOKEEPER_HOME/bin
~~~

> 4/ 加载环境配置：

~~~
source /etc/profile
~~~

> 5/ 修改完成后切换回hadoop用户：

~~~
su - hadoop
~~~

#### 2.3 修改Zookeeper配置文件

> 1、用root用户操作

~~~
cd zookeeper/conf
cp zoo_sample.cfg zoo.cfg
~~~
 
> 2、vi zoo.cfg
 
> 3、添加内容：

~~~
dataDir=/root/apps/zookeeper/zkdata
dataLogDir=/home/hadoop/zookeeper/log
server.1=mini1:2888:3888     ## (心跳端口、选举端口)
server.2=mini2:2888:3888
server.3=mini3:2888:3888
~~~
 
> 4、创建文件夹：

~~~
cd /home/hadoop/zookeeper/
mkdir zkdata
mkdir -m 755 log
~~~
 
> 5、在data文件夹下新建myid文件，myid的文件内容为：

~~~
cd zkdata
echo 1 > myid
~~~

#### 2.4 分发安装包到其他机器

~~~
scp -r /root/apps root@mini2:/root/
scp -r /root/apps root@mini3:/root/
~~~

#### 2.5 修改其他机器的配置文件

~~~
到mini2上：修改myid为：2
到mini3上：修改myid为：3
~~~

#### 2.6 启动（每台机器）

> 注：
> 1、事先将三台服务器的防火墙都关掉
> 2、全网统一hosts映射
> 先配好一台上的hosts
> 然后：

~~~
scp  /etc/hosts  mini2:/etc
scp  /etc/hosts  mini3:/etc
~~~

> 3、然后一台一台地启动

~~~
bin/zkServer.sh start
~~~

 
 
~~~
或者编写一个脚本来批量启动所有机器：
for host in "mini1 mini2 mini3"
do
ssh $host "source/etc/profile;/root/apps/zookeeper/bin/zkServer.sh start"
~~~
#### 2.7 查看集群状态

~~~
1、jps（查看进程）
2、zkServer.sh status（查看集群状态，主从信息）
~~~


## 4 Zookeeper核心工作机制

### 1 zookeeper特性

1. Zookeeper：一个leader，多个follower组成的集群
2. 全局数据一致：每个server保存一份相同的数据副本，client无论连接到哪个server，数据都是一致的
3. 分布式读写，更新请求转发，由leader实施
4. 更新请求顺序进行，来自同一个client的更新请求按其发送顺序依次执行
5. 数据更新原子性，一次数据更新要么成功（半数以上节点成功），要么失败
6. 实时性，在一定时间范围内，client能读到最新数据
 
### 2 zookeeper数据结构

#### 2.1 概况

1. 层次化的目录结构，命名符合常规文件系统规范(见下图)
2. 每个节点在zookeeper中叫做znode,并且其有一个唯一的路径标识
3. 节点Znode可以包含数据(只能存储很小量的数据，<1M;最好是1k字节以内)和子节点（但是EPHEMERAL类型的节点不能有子节点，下一页详细讲解）
4. 客户端应用可以在节点上设置监视器（后续详细讲解） 

#### 2.2 数据结构图

![](https://note.youdao.com/yws/api/personal/file/AB4834A4890E455C9AE56D7E64E7869D?method=download&shareKey=3b6eacf2728da945c7435f1a27a62726)
 
wps1A0D
 
 
 
#### 2.3 节点类型

> 1、Znode有两种类型：

~~~
短暂（ephemeral）（断开连接自己删除）
持久（persistent）（断开连接不删除）
~~~

> 2、Znode有四种形式的目录节点（默认是persistent ）

~~~
PERSISTENT
PERSISTENT_SEQUENTIAL（持久序列/test0000000019 ）
EPHEMERAL
EPHEMERAL_SEQUENTIAL
~~~
> 3、创建znode时设置顺序标识，znode名称后会附加一个值，顺序号是一个单调递增的计数器，由父节点维护
> 4、在分布式系统中，顺序号可以被用于为所有的事件进行全局排序，这样客户端可以通过顺序号推断事件的顺序
 
 
## 5 原理补充

> Zookeeper虽然在配置文件中并没有指定master和slave
> 但是，zookeeper工作时，是有一个节点为leader，其他则为follower
> Leader是通过内部的选举机制临时产生的

 ### 1 zookeeper的选举机制（zk的数据一致性核心算法paxos）
 
> 以一个简单的例子来说明整个选举的过程.
> 假设有五台服务器组成的zookeeper集群,它们的id从1-5,同时它们都是最新启动的,也就是没有历史数据,在存放数据量这一点上,都是一样的.假设这些服务器依序启动,来看看会发生什么.


1) 服务器1启动,此时只有它一台服务器启动了,它发出去的报没有任何响应,所以它的选举状态一直是LOOKING状态
2) 服务器2启动,它与最开始启动的服务器1进行通信,互相交换自己的选举结果,由于两者都没有历史数据,所以id值较大的服务器2胜出,但是由于没有达到超过半数以上的服务器都同意选举它(这个例子中的半数以上是3),所以服务器1,2还是继续保持LOOKING状态.
3) 服务器3启动,根据前面的理论分析,服务器3成为服务器1,2,3中的老大,而与上面不同的是,此时有三台服务器选举了它,所以它成为了这次选举的leader.
4) 服务器4启动,根据前面的分析,理论上服务器4应该是服务器1,2,3,4中最大的,但是由于前面已经有半数以上的服务器选举了服务器3,所以它只能接收当小弟的命了.
5) 服务器5启动,同4一样,当小弟.

 
### 9.2 非全新集群的选举机制(数据恢复)

> 那么，初始化的时候，是按照上述的说明进行选举的，但是当zookeeper运行了一段时间之后，有机器down掉，重新选举时，选举过程就相对复杂了。
> 需要加入数据version、leader id和逻辑时钟。
> 数据version：数据新的version就大，数据每次更新都会更新version。
> Leader id：就是我们配置的myid中的值，每个机器一个。
> 逻辑时钟：这个值从0开始递增,每次选举对应一个值,也就是说:  如果在同一次选举中,那么这个值应该是一致的 ;  逻辑时钟值越大,说明这一次选举leader的进程更新.
> 选举的标准就变成：

               

1. 逻辑时钟小的选举结果被忽略，重新投票
2. 统一逻辑时钟后，数据id大的胜出
3. 数据id相同的情况下，leader id大的胜出

> 根据这个规则选出leader。

 
 
 
 
