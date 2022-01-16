### Docker 高级篇

### 一、Docker复杂安装

#### 1.1 安装mysql主从复制

##### 1.1.1 主从复制原理

```properties
默认你已经了解
```

##### 1.1.2 主从搭建步骤

>1、新建主服务器容器实例3307
>
>```shell
>docker run -p 3307:3306 --name mysql-master \ 
>-v /mydata/mysql-master/log:/var/log/mysql \ 
>-v /mydata/mysql-master/data:/var/lib/mysql \ 
>-v /mydata/mysql-master/conf:/etc/mysql \ 
>-e MYSQL_ROOT_PASSWORD=root  \ 
>-d mysql:5.7 
>```
>
>2、进入/mydata/mysql-master/conf目录下新建my.cnf
>
>```shell
>[mysqld] 
>## 设置server_id，同一局域网中需要唯一 
>server_id=101  
>## 指定不需要同步的数据库名称 
>binlog-ignore-db=mysql   
>## 开启二进制日志功能 
>log-bin=mall-mysql-bin   
>## 设置二进制日志使用内存大小（事务） 
>binlog_cache_size=1M   
>## 设置使用的二进制日志格式（mixed,statement,row） 
>binlog_format=mixed   
>## 二进制日志过期清理时间。默认值为0，表示不自动清理。 
>expire_logs_days=7   
>## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。 
>## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致 
>slave_skip_errors=1062 
>```
>
>3、修改完配置后重启master实例
>
>```shell
>docker restart mysql-master
>```
>
>4、进入mysql-master容器
>
>```shell
>docker exec -it mysql-master /bin/bash
>
>mysql -uroot -proot
>```
>
>5、maser容器实例内创建数据同步用户
>
>```shell
># 创建同步用户
>CREATE USER 'slave'@'%' IDENTIFIED BY '123456';
>
># 同步用户授权
>GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';
>```
>
>6、新建从服务容器实例3308
>
>```shell
>docker run -p 3308:3306 --name mysql-slave \ 
>-v /mydata/mysql-slave/log:/var/log/mysql \ 
>-v /mydata/mysql-slave/data:/var/lib/mysql \ 
>-v /mydata/mysql-slave/conf:/etc/mysql \ 
>-e MYSQL_ROOT_PASSWORD=root  \ 
>-d mysql:5.7 
>```
>
>7、进入/mydata/mysql-slave/conf目录下新建my.cnf
>
>```shell
># 编辑my.cnf
>vim my.cnf
># 添加配置文件
>[mysqld] 
>## 设置server_id，同一局域网中需要唯一 
>server_id=102 
>## 指定不需要同步的数据库名称 
>binlog-ignore-db=mysql   
>## 开启二进制日志功能，以备Slave作为其它数据库实例的Master时使用 
>log-bin=mall-mysql-slave1-bin   
>## 设置二进制日志使用内存大小（事务） 
>binlog_cache_size=1M   
>## 设置使用的二进制日志格式（mixed,statement,row） 
>binlog_format=mixed   
>## 二进制日志过期清理时间。默认值为0，表示不自动清理。 
>expire_logs_days=7   
>## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。 
>## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致 
>slave_skip_errors=1062   
>## relay_log配置中继日志 
>relay_log=mall-mysql-relay-bin   
>## log_slave_updates表示slave将复制事件写进自己的二进制日志 
>log_slave_updates=1   
>## slave设置为只读（具有super权限的用户除外） 
>read_only=1 
>
>```
>
>8、修改完配置后重启slave实例
>
>```shell
>docker restart mysql-slave
>```
>
>9、在主数据库中查看主从同步状态
>
>```shell
>show master staus;
>```
>
>10、进入mysql-slave容器
>
>```shell
>docker exec -it mysql-slave /bin/bash
>mysql -uroot -proot
>```
>
>11、在从数据库中配置主从复制
>
>```shell
>change master to master_host='宿主机ip', master_user='slave', master_password='123456', master_port=3307, master_log_file='mall-mysql-bin.000001', master_log_pos=617, master_connect_retry=30; 
>```
>
>![1](images/1.png)
>
>12、在从数据库中查看主从同步状态
>
>```shell
>show slave status\G;
>```
>
>13、在从数据库中开启主从同步
>
>```shell
>start slave;
>```
>
>14、查看从数据库状态发现已经同步
>
>![2](images/2.png)
>
>15、主从复制测试
>
>```properties
>1.主机新建数据库 --->  使用数据库 ---> 新建表 --->插入数据 ， ok
>2.从机使用库 ---> 查看记录 ok
>```
>
>

#### 1.2 安装redis集群(大厂面试题第4季-分布式存储案例真题)

##### 1.2.1 cluster(集群)模式-docker版哈希槽分区进行亿级数据存储

>一、面试题
>
>问题：1~2亿条数据需要缓存，请问如何设计这个存储案例
>
> 回答：单机单台100%不可能，肯定是分布式存储，用redis如何落地？
>
>上述问题阿里P6~P7工程案例和场景设计类必考题目，
>一般业界有3种解决方案
>1. 哈希取余分区
>
>```shell
>2亿条记录就是2亿个k,v，我们单机不行必须要分布式多机，假设有3台机器构成一个集群，用户每次读写操作都是根据公式：
>hash(key) % N个机器台数，计算出哈希值，用来决定数据映射到哪一个节点上。 
>
>优点：
>  简单粗暴，直接有效，只需要预估好数据规划好节点，例如3台、8台、10台，就能保证一段时间的数据支撑。使用Hash算法让固定的一部分请求落到同一台服务器上，这样每台服务器固定处理一部分请求（并维护这些请求的信息），起到负载均衡+分而治之的作用。 
>  
>缺点：
>   原来规划好的节点，进行扩容或者缩容就比较麻烦了额，不管扩缩，每次数据变动导致节点有变动，映射关系需要重新进行计算，在服务器个数固定不变时没有问题，如果需要弹性扩容或故障停机的情况下，原来的取模公式就会发生变化：Hash(key)/3会变成Hash(key) /?。此时地址经过取余运算的结果将发生很大变化，根据公式获取的服务器也会变得不可控。 
>某个redis机器宕机了，由于台数数量变化，会导致hash取余全部数据重新洗牌。 
>
>```
>
>2. 一致性哈希算法分区
>
>   1、是什么
>   一致性Hash算法背景 
>   　一致性哈希算法在1997年由麻省理工学院中提出的，设计目标是为了解决 
>   分布式缓存数据 变动和映射问题 ，某个机器宕机了，分母数量改变了，自然取余数不OK了。
>   2、能干什么
>   提出一致性Hash解决方案。目的是当服务器个数发生变动时，尽量减少影响客户端到服务器的映射关系。
>   3、3大步骤
>   【算法构建一致性哈希环】
>       一致性哈希算法必然有个hash函数并按照算法产生hash值，这个算法的所有可能哈希值会构成一个全量集，这个集合可以成为一个hash空间[0,2^32-1]，这个是一个线性空间，但是在算法中，我们通过适当的逻辑控制将它首尾相连(0 = 2^32),这样让它逻辑上形成了一个环形空间。 
>
>   它也是按照使用取模的方法，前面笔记介绍的节点取模法是对节点（服务器）的数量进行取模。而一致性Hash算法是对2^32取模，简单来说， 一致性Hash算法将整个哈希值空间组织成一个虚拟的圆环 ，如假设某哈希函数H的值空间为0-2^32-1（即哈希值是一个32位无符号整形），整个哈希环如下图：整个空间 按顺时针方向组织 ，圆环的正上方的点代表0，0点右侧的第一个点代表1，以此类推，2、3、4、……直到2^32-1，也就是说0点左侧的第一个点代表2^32-1， 0和2^32-1在零点中方向重合，我们把这个由2^32个点组成的圆环称为Hash环。
>
>![3](images/3.png)
>
>【服务器IP节点映射】
>   将集群中各个IP节点映射到环上的某一个位置。 
>   将各个服务器使用Hash进行一个哈希，具体可以选择服务器的IP或主机名作为关键字进行哈希，这样每台机器就能确定其在哈希环上的位置。假如4个节点NodeA、B、C、D，经过IP地址的 哈希函数 计算(hash(ip))，使用IP地址哈希后在环空间的位置如下：
>
>![4](images/4.png)
>
>【key落到服务器的落键规则】
>	当我们需要存储一个kv键值对时，首先计算key的hash值，hash(key)，将这个key使用相同的函数Hash计算出哈希值并确定此数据在环上的位置， 从此位置沿环顺时针“行走” ，第一台遇到的服务器就是其应该定位到的服务器，并将该键值对存储在该节点上。 
>如我们有Object A、Object B、Object C、Object D四个数据对象，经过哈希计算后，在环空间上的位置如下：根据一致性Hash算法，数据A会被定为到Node A上，B被定为到Node B上，C被定为到Node C上，D被定为到Node D上。
>
>![5](images/5.png)
>
>4、优点
>
>一致性哈希算法的**容错性**
>
>```properties
>假设Node C宕机，可以看到此时对象A、B、D不会受到影响，只有C对象被重定位到Node D。一般的，在一致性Hash算法中，如果一台服务器不可用，则 受影响的数据仅仅是此服务器到其环空间中前一台服务器（即沿着逆时针方向行走遇到的第一台服务器）之间数据 ，其它不会受到影响。简单说，就是C挂了，受到影响的只是B、C之间的数据，并且这些数据会转移到D进行存储。 
>```
>
>![6](images/6.png)
>
>一致性哈希算法的**扩展性**
>
>```properties
>数据量增加了，需要增加一台节点NodeX，X的位置在A和B之间，那收到影响的也就是A到X之间的数据，重新把A到X的数据录入到X上即可， 
>不会导致hash取余全部数据重新洗牌。 
>```
>
>![7](images/7.png)
>
>5、缺点
>
>一致性哈希算法的数据倾斜问题
>
>```properties
>一致性Hash算法在服务 节点太少时 ，容易因为节点分布不均匀而造成 数据倾斜 （被缓存的对象大部分集中缓存在某一台服务器上）问题， 
>例如系统中只有两台服务器：
>```
>
>![8](images/8.png)
>
>6、小总结
>
>```properties
>为了在节点数目发生改变时尽可能少的迁移数据 
>  
>将所有的存储节点排列在收尾相接的Hash环上，每个key在计算Hash后会 顺时针 找到临近的存储节点存放。 
>而当有节点加入或退出时仅影响该节点在Hash环上 顺时针相邻的后续节点 。   
>  
>优点 
>加入和删除节点只影响哈希环中顺时针方向的相邻的节点，对其他节点无影响。 
>  
>缺点  
>数据的分布和节点的位置有关，因为这些节点不是均匀的分布在哈希环上的，所以数据在进行存储时达不到均匀分布的效果。 
>
>```

> 3. 哈希槽分区
>
> ```properties
> 1. 为什么出现
> 一致性哈希算法的数据倾斜问题
> 哈希槽是指就是一个数组，数组[0,2^14-1]形成的hash slot空间。
> 2. 能干什么
> 解决均匀分配的问题，在数据和节点之间有加入了一层，把这层称为哈希槽(slot)，用于管理数据和节点之间的关系，现在就相当于节点上放的是槽，槽里放的是数据。
> 
> ```
>
> ![9](images/9.png)
>
> ```properties
> 槽解决的是粒度问题，相当于把粒度变大了，这样便于数据移动。 
> 哈希解决的是映射问题，使用key的哈希值来计算所在的槽，便于数据分配。
> 3. 多少个hash槽 
> 一个集群只能有16384个槽，编号0-16383（0-2^14-1）。这些槽会分配给集群中的所有主节点，分配策略没有要求。可以指定哪些编号的槽分配给哪个主节点。集群会记录节点和槽的对应关系。解决了节点和槽的关系后，接下来就需要对key求哈希值，然后对16384取余，余数是几key就落入对应的槽里。slot = CRC16(key) % 16384。以槽为单位移动数据，因为槽的数目是固定的，处理起来比较容易，这样数据移动问题就解决了。 
> ```
>
> 

##### 1.2.2 redis集群3主3从扩缩容配置案例

>一、关闭防火墙+启动docker后台服务
>
>```shell
>systemctl start docker
>```
>
>二、新建6个docker容器redis实例
>
>```shell
># 创建并运行docker容器实例
>docker run 
># 容器名字
>--name redis-node-6
># 使用宿主机的IP和端口，默认
>--net host
># 获取宿主机root用户权限
>--privileged=true
># 容器卷，宿主机地址:docker内部地址
>-v /data/redis/share/redis-node-6:/data
># redis镜像和版本号
>redis:6.0.8
># 开启redis集群
>--cluster-enabled yes 
># 开启持久化
>--applendonly yes 
>```
>
>```shell
>docker run -d --name redis-node-1 --net host --privileged=true -v /data/redis/share/redis-node-1:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6381 
>  
>docker run -d --name redis-node-2 --net host --privileged=true -v /data/redis/share/redis-node-2:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6382 
>  
>docker run -d --name redis-node-3 --net host --privileged=true -v /data/redis/share/redis-node-3:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6383 
>  
>docker run -d --name redis-node-4 --net host --privileged=true -v /data/redis/share/redis-node-4:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6384 
>  
>docker run -d --name redis-node-5 --net host --privileged=true -v /data/redis/share/redis-node-5:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6385 
>  
>docker run -d --name redis-node-6 --net host --privileged=true -v /data/redis/share/redis-node-6:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6386 
>```
>
>三、进入容器redis-node-1并为6台机器构建集群关系
>
>```shell
># 进入容器
>docker exec -it redis-node-1 /bin/bash
>```
>
>//注意，进入docker容器后才能执行一下命令，且注意自己的真实IP地址 
>
>```shell
>redis-cli --cluster create 192.168.111.147:6381 192.168.111.147:6382 192.168.111.147:6383 192.168.111.147:6384 192.168.111.147:6385 192.168.111.147:6386 --cluster-replicas 1 
>```
>
>--cluster-replicas 1 表示为每个master创建一个slave节点 
>
>![10](images/10.png)
>
>![11](images/11.png)
>
>四、连接进入6318作为切入点，查看集群状态
>
>![12](images/12.png)
>
>```
>cluster info
>
>cluster nodes
>```
>
>

##### 1.2.3 主从容错切换迁移案例

>一、数据读写存储
>
>启动6机构成的集群并通过exec进入
>
>对6381新增两个key
>
>防止路由时效加参数-c并新增连个key
>
>![13](images/13.png)
>
>查看集群信息
>
>```shell
>redis-cli --cluster check 192.168.111.147:6381 
>```
>
>![14](images/14.png)
>
>二、容错切换迁移
>
>1. 主6381和从机切换，先停止主机6381
>
>```properties
>6381主机停了，对应的真实从机上位
>6381作为1号主机分配的从机以实际情况为准，具体是几号机器就是几号
>```
>
>2. 再次查看集群信息
>
>   ![15](images/15.png)
>
>   6381宕机了，6385上位成为了新的master。 
>
>   备注：本次脑图笔记6381为主下面挂从6385 。 
>
>   每次案例下面挂的从机以实际情况为准，具体是几号机器就是几号 
>
>3. 先还原之前的3主3从
>
>```shell
># 先启6381
>docker start redis-node-1
># 再停6385 
>docker stop redis-node-5
># 再起6385
>docker start redis-node-5
>主从机器分配情况一实际情况为准
>```
>
>4. 查看集群状态
>
>   ```shell
>   redis-cli --cluster check 自己IP:6381
>   ```
>
>   ![16](images/16.png)

##### 1.2.4 主从扩容案例

>一、新建6387、6388两个节点+新建后启动+查看是否8节点
>
>```shell
>docker run -d --name redis-node-7 --net host --privileged=true -v /data/redis/share/redis-node-7:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6387 
>
>docker run -d --name redis-node-8 --net host --privileged=true -v /data/redis/share/redis-node-8:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6388 
>
>docker ps 
>```
>
>二、进入6387容器实例内部
>
>```shell
>docker exec -it redis-node-7 /bin/bash
>```
>
>三、将新增的6387节点(空槽号)作为master节点加入原集群
>
>```shell
>将新增的6387作为master节点加入集群
>redis-cli --cluster  add-node  自己实际IP地址: 6387  自己实际IP地址: 6381 
>6387 就是将要作为master新增节点 
>6381 就是原来集群节点里面的领路人，相当于6387拜拜6381的码头从而找到组织加入集群 
>```
>
>四、检查集群情况第1次
>
>```shell
>redis-cli --cluster check 真实ip地址:6381 
># 例如
>redis-cli --cluster check 192.168.111.147:6381 
>```
>
>五、重新分派槽号
>
>```shell
>重新分派槽号
>命令:redis-cli --cluster  reshard  IP地址:端口号 
>redis-cli --cluster reshard 192.168.111.147:6381 
>```
>
>![17](images/17.png)
>
>六、检查集群情况第2次
>
>```shell
>redis-cli --cluster check 真实ip地址:6381 
>
>为什么6387是3个新的区间，以前的还是连续？
>重新分配成本太高，所以前3家各自匀出来一部分，从6381/6382/6383三个旧节点分别匀出1364个坑位给新节点6387 
>```
>
>![18](images/18.png)
>
>七、为主节点6387分配从节点6388
>
>```shell
>命令：redis-cli  --cluster add-node  ip:新slave端口 ip:新master端口 --cluster-slave --cluster-master-id 新主机节点ID
> 
>redis-cli --cluster add-node 192.168.111.147:6388 192.168.111.147:6387 --cluster-slave --cluster-master-id e4781f644d4a4e4d4b4d107157b9ba8144631451-------这个是6387的编号，按照自己实际情况 
>```
>
>![20](images/20.png)
>
>八、检查集群情况第3次
>
>```shell
>redis-cli --cluster check 192.168.111.147:6382 
>```
>
>![19](images/19.png)

##### 1.2.5 主从缩容案例

>一、目的：6387和6388下线
>
>二、检查集群情况1获得6388的节点ID
>
>```shell
>redis-cli --cluster check 192.168.111.147:6382 
>```
>
>三、将6388删除  从集群中将4号从节点6388删除
>
>```shell
>命令：redis-cli --cluster  del-node  ip:从机端口 从机6388节点ID
>  
>redis-cli --cluster  del-node  192.168.111.147:6388 5d149074b7e57b802287d1797a874ed7a1a284a8 
>
>redis-cli --cluster check 192.168.111.147:6382 
>
>检查一下发现，6388被删除了，只剩下7台机器了。
>```
>
>四、将6387的槽号清空，重新分配，本例将清出来的槽号都给6381
>
>```shell
>redis-cli --cluster reshard 192.168.111.147:6381 
>```
>
>![21](images/21.png)
>
>五、检查集群情况第二次
>
>```shell
>redis-cli --cluster check 192.168.111.147:6381
>  
>4096个槽位都指给6381，它变成了8192个槽位，相当于全部都给6381了，不然要输入3次，一锅端 
>```
>
>![22](images/22.png)
>
>六、将6387删除
>
>```shell
># 命令：redis-cli --cluster del-node ip:端口 6387节点ID
>redis-cli --cluster del-node 192.168.111.147:6387 e4781f644d4a4e4d4b4d107157b9ba8144631451 
>```
>
>七、检查集群情况第三次
>
>```shell
>redis-cli --cluster check 192.168.111.147:6381 
>```
>
>![23](images/23.png)

### 二、DockerFile解析

#### 2.1 DockerFile是什么

DockerFile是用来构建Docker镜像的文本文件，是有一条条构建镜像所需的指令和参数构成的脚本。

![24](images/24.png)

官网：https://docs.docker.com/engine/reference/builder/

构建三步骤

````shell
1、编写DockerFile文件
2、docker build命令构建镜像
3、docker run 依镜像运行容器实例
````

#### 2.2 DockerFile构建过程解析

DockerFile内容基础知识

```shell
1. 每条保留字指令都必须为大写字母且后面跟随至少一个参数
2. 指令按照从上到下，顺序执行
3. #表示注释
4. 每条指令都会创建一个新的镜像层并对镜像进行提交。
```

Docker执行DockerFile的大致流程

```
1. docker从技术镜像运行一个容器
2. 执行一条指令比鞥对容器做出修改
3. 执行类似docker commit 的操作提交一个新的镜像层
4. docker 在基于刚提交的镜像运行一个新容器
5. 执行dockerfile中的下一条指令直到所有执行执行完成。
```

#### 2.3 小总结

>从应用软件的角度来看，Dockerfile、Docker镜像与Docker容器分别代表软件的三个不同阶段， 
>
> Dockerfile是软件的原材料 
>
>Docker镜像是软件的交付品 
>
>Docker容器则可以认为是软件镜像的运行态，也即依照镜像运行的容器实例 
>
>Dockerfile面向开发，Docker镜像成为交付标准，Docker容器则涉及部署与运维，三者缺一不可，合力充当Docker体系的基石。 
>
>![25](images/25.png)
>
>1. Dockerfile，需要定义一个Dockerfile，Dockerfile定义了进程需要的一切东西。Dockerfile涉及的内容包括执行代码或者是文件、环境变量、依赖包、运行时环境、动态链接库、操作系统的发行版、服务进程和内核进程(当应用进程需要和系统服务和内核进程打交道，这时需要考虑如何设计namespace的权限控制)等等; 
>
>2. Docker镜像，在用Dockerfile定义一个文件之后，docker build时会产生一个Docker镜像，当运行 Docker镜像时会真正开始提供服务; 
>
>3. Docker容器，容器是直接提供服务的。 

#### 2.4 DockerFile常用保留字指令

>1. 参考tomcat8的dockerfile入门
>
>https://github.com/docker-library/tomcat
>
>2. From
>
>   ```properties
>   基础镜像，当前新镜像是基于哪个镜像的，指定一个已经存在的镜像作为模板，第一条必须是from
>   ```
>
>3. MANINTAINER
>
>   镜像维护者的姓名和邮箱地址
>
>4. Run
>
>   容器构建时需要运行的命令
>
>   两种格式：
>
>   shell格式
>
>   ```shell
>   <命令行命令>等同于，在终端操作的shell命令
>   
>   RUN yum -y install vim
>   ```
>
>   exec格式
>
>   ![26](images/26.png)
>
>   RUN是在docker build时运行
>
>5. EXPOSE
>
>   当前容器对外暴露出的端口
>
>6. WORKDIR
>
>   指定在创建容器后。终端默认登录的进来工作目录，一个落脚点。
>
>7. USER
>
>   指定该镜像以什么样的用户去执行，如果都不指定，默认是root
>
>8. ENV
>
>   用来在构建镜像过程中设置环境变量
>
>   ```properties
>   ENV MY_PATH /usr/mytest 
>   这个环境变量可以在后续的任何RUN指令中使用，这就如同在命令前面指定了环境变量前缀一样； 
>   也可以在其它指令中直接使用这些环境变量， 
>     
>   比如：WORKDIR $MY_PATH 
>   ```
>
>   
>
>9. ADD
>
>   将宿主机目录下的文件拷贝进镜像且会自动处理URL和解压tar压缩包
>
>10. COPY
>
>    类似ADD，拷贝文件和目录到镜像中。将从构建上下文目录中<源路径>的文件/目录复制到新的一层镜像内的<目标路径>位置
>
>    ```shell
>    COPY src dest
>    
>    COPY["src","dest"]
>    
>    <src源路径>：源文件或源目录
>    
>    <dest目标路径>: 容器内的指定路径，该路径不用事先建好，路径不存在的话，会自动创建。
>    ```
>
>    
>
>11. VOLUME
>
>    容器数据卷，用于数据保存和持久化的工作
>
>12. CMD
>
>    指定容器启动后的要干的事情。
>
>    【注意】
>
>    ```shell
>    Dockerfile 中可以由多个CMD指令，但是只有最后一个生效，CMD会被docker run 之后的参数替换。
>    ```
>
>    参考官网Tomcat的dockerfile演示讲解
>
>    官网最后一行命令
>
>    ```
>    EXPOSE 8080
>    CMD ["catalina.sh","run"]
>    ```
>
>    我们演示自己的覆盖操作
>
>    ```shell
>    docker run -it -p 8080:8080  容器ID /bin/bash
>    ```
>
>    他和前面RUN命令的区别
>
>    ```shell
>    CMD 是在 docker run 时运行。
>    
>    RUN 是在docker build 时运行
>    ```
>
>    
>
>13. ENTRYPOINT 
>
>    1. 也是用来指定一个容器启动时要运行的命令
>
>    2. 类似于CMD指令，但是ENTRYPOINT不会被docker run 后面的命令覆盖，而且这些命令行参数会被当作参数送给ENTRYPOINT指令指定的程序。
>
>    3. 命令格式和案例说明
>
>       ```shell
>       命令格式：ENTRYPOINT["<executeable>","<param1>","<param2>",...]
>       
>       ENTRYPOINT 可以和CMD一起用，一般是 变参 才会使用 CMD ，这里的CMD等于是在给 ENTRYPOINT 传参。当制定了 ENTRYPOINT 后，CMD的含义就发生了变化，不再是直接运行其命令而是将 CMD 的内容作为参数传递给 ENTRYPOINT 指定，他两个组合会变成<ENTRYPOINT> "<CMD>"
>       案例如下：假设已通过 Dockerfile 构建了 nginx:test 镜像
>       ```
>
>        ![27](images/27.png)
>
>       | 是否传参         | 按照dockerfile编写执行         | 传参运行                                      |
>       | ---------------- | ------------------------------ | --------------------------------------------- |
>       | Docker命令       | docker run nginx:test          | docker run nginx:test -c /etc/nginx/ new.conf |
>       | 衍生出的实际命令 | nginx -c /etc/nginx/nginx.conf | nginx -c /etc/nginx/ new.conf                 |
>
>        
>
>    优点：在执行docker run 的时候可以指定 ENTRYPOINT 运行所需的参数。
>
>    注意：如果Dockerfile 中如果存在多个 ENTRYPOINT 指令，进最后一个生效。
>
>    14. 小总结
>
>        ![28](images/28.png)

#### 2.5 案例

##### 2.5.1 自定义镜像mycentosjava8

**要求**

```shell
Centos7镜像具备 vim + ifconfig + jdk8

JDK下载镜像地址
官网：https://www.oracle.com/java/technologies/downloads/#java8 
https://mirrors.yangxingzhen.com/jdk/
```

**编写**

```shell
准备编写Dockerfile文件 
【注意】大写字母D

FROM centos
MAINTAINER zzyy<zzyybs@126.com> 
  
ENV MYPATH /usr/local 
WORKDIR $MYPATH 
  
#安装vim编辑器 
RUN yum -y install vim 
#安装ifconfig命令查看网络IP 
RUN yum -y install net-tools 
#安装java8及lib库 
RUN yum -y install glibc.i686 
RUN mkdir /usr/local/java 
#ADD 是相对路径jar,把jdk-8u171-linux-x64.tar.gz添加到容器中,安装包必须要和Dockerfile文件在同一位置 
ADD jdk-8u171-linux-x64.tar.gz /usr/local/java/ 
#配置java环境变量 
ENV JAVA_HOME /usr/local/java/jdk1.8.0_171 
ENV JRE_HOME $JAVA_HOME/jre 
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH 
ENV PATH $JAVA_HOME/bin:$PATH 
  
EXPOSE 80 
 
CMD echo $MYPATH 
CMD echo "success--------------ok" 
CMD /bin/bash 
```

**构建**

```shell
docker build -t 新镜像名字: TAG

例如：docker build -t centosjava8:1.5 .

【注意】
上面TAG 后面有个空格，有个点
```

**运行**

```shell
docker run -it 新镜像名字:TAG

docker run -it centosjava8:1.5 /bin/bash 
```

![29](images/29.png)

**再体会下UnionFS（联合文件系统）**

```properties
UnionFS（联合文件系统）：Union文件系统（UnionFS）是一种分层、轻量级并且高性能的文件系统，它支持 对文件系统的修改作为一次提交来一层层的叠加， 同时可以将不同目录挂载到同一个虚拟文件系统下(unite several directories into a single virtual filesystem)。Union 文件系统是 Docker 镜像的基础。 镜像可以通过分层来进行继承 ，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。

特性：一次同时加载多个文件系统，但从外面看起来，只能看到一个文件系统，联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录 
```

##### 2.5.2 虚悬镜像

**是什么**

```
仓库名，标签都是 <none> 的镜像，俗称dangling image

Dockerfile 写一个
```

![30](images/30.png)

**查看**

```shell
docker image ls -f dangling=true
命令结果如下图：
```

![31](images/31.png)

**删除**

```
docker image prune 

虚悬镜像已经市区存在价值，可以删除
```

![32](images/32.png)

##### 2.5.3 家庭作业自定义myubuntu

````shell
# 编写
准备编写DockerFile文件
vim Dockerfile
----------------------
FROM ubuntu
MAINTAINER zzyy<zzyybs@126.com> 
  
ENV MYPATH /usr/local 
WORKDIR $MYPATH 
  
RUN apt-get update 
RUN apt-get install net-tools 
#RUN apt-get install -y iproute2 
#RUN apt-get install -y inetutils-ping 
  
EXPOSE 80 
  
CMD echo $MYPATH 
CMD echo "install inconfig cmd into ubuntu success--------------ok" 
CMD /bin/bash 
------------------------
# 构建
docker build -t 新镜像名字:TAG

#运行
docker run -it 新镜像名字:TAG

````

















