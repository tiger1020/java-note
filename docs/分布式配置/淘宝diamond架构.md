# 1.diamond架构

<img src="/images/分布式配置-diamond架构.png">

diamond的特点是简单、可靠、易用：

简单：整体结构非常简单，从而减少了出错的可能性。

易用：客户端使用只需要两行代码，暴露的接口都非常简单，易于理解。

# 2.diamond流程

<img src="/images/分布式配置-diamond流程.png">

## 2.1.server集群数据同步

diamond-server将数据存储在mysql和本地文件中，mysql是一个中心，diamond认为存储在mysql中的数据绝对正确，除此之外，server会将数据存储在本地文件中。

同步数据有两种方式：

（1）server写数据时，先将数据写入mysql，然后写入本地文件，写入完成后发送一个HTTP请求给集群中的其他server，其他server收到请求，从mysql中dump刚刚写入的数据至本地文件。

（2）server启动后会启动一个**定时任务**，定时从mysql中dump所有数据至本地文件。

## 2.2.client获取server地址

server地址存储在一台具有域名的机器上的HTTP server中，我们称它为地址服务器

## 2.3.client主动获取数据

client调用getAvailableConfigInfomation()， 即可获取一份最新的可用的配置数据，获取过程实际上是拼接http url，使用http-client调用http method的过程。

为了避免短时间内大量的获取数据请求发向server，client端实现了一个带有过期时间的缓存，client将本次获取到的数据保存在缓存中，在过期时间内的所有请求，都返回缓存内的数据，不向server发出请求。

## 2.4.client运行中感知数据变化

这个特性是通过比较client和server的数据的MD5值实现的。

**server端**：server在启动时，会将所有数据的MD5加载到内存中（MD5根据某算法得出，保证数据内容不同，MD5不同，MD5存储在mysql中），数据更新时，会更新内存中对应的MD5

**client端**：client在启动并第一次获取数据后，会将数据的MD5保存在内存中，并且在启动时会启动一个**定时任务**，定时去server检查数据是否变化。每次检查时，client将MD5传给server，server比较传来的MD5和自身内存中的MD5是否相同；

> 如果相同，说明数据没变，返回一个标示数据不变的字符串给client；
>
> 如果不同，说明数据变了，返回变化数据的dataId和group给client. client收到变化数据的dataId和group，再去server请求一次数据，拿回数据后回调监听器。

# 3.diamond容灾机制

## 3.1.server存储数据的方式

server存储数据是“数据库 + 本地文件”的方式

## 3.2.server可以是一个集群

集群中的一台server不可用了，client发现后可以自动切换到其他server上进行访问，自动切换在client内部实现

## 3.3.client保存snapshot

client每次从server获取到数据后，都会将数据保存在本地文件系统，diamond称之为snapshot，即数据快照。当client下次启动发现在超时时间内所有server均不可用（可能是网络故障），它会使用snapshot中的数据快照进行启动。

## 3.4.client与server可以完全分离

client可以和server完全分离，单独使用，diamond定义了一个“容灾目录”的概念，client在启动时会创建这个目录，每次主动获取数据（即调用getAvailableConfigInfomation()方法），都会优先从“容灾目录”获取数据，**如果client按照一个固定的规则，手动配置**，在“容灾目录”下配置了需要的数据，那么client直接获取到数据返回，不再通过网络从diamond-server获取数据。同样的，在每次轮询时，都会优先轮询“容灾目录”，如果发现配置还存在于其中，则不再向server发出轮询请求。 以上的情形， 会持续到“容灾目录”的配置数据被删除为止。

> **根据以上的容灾机制，我们可以总结一下diamond整个系统完全不可用的条件：**
>
> 1、mysql数据库不可用。
>
> 2、所有diamond-server均不可用。
>
> 3、client主动删除了snapshot本地磁盘文件。
>
> 4、client没有备份配置数据，导致其不能配置“容灾目录”。
>
> 同时满足以上4个条件的概率，在生产环境中是极小的。

