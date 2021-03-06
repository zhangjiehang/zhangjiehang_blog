---
title: redis数据持久化
categories: Redis
---
# 浅谈Redis数据持久化方式以及问题
<!--more-->
---
文章参考[http://redis.io/topics/persistence]

---

## Redis持久化方式
1. RBD持久化：在指定时间内生成当前数据集的快照
2. AOF持久化：记录当前服务器执行的所有写操作，并在服务器启动时，通过重新执行这些命令来还原数据（**ps:搭建了集群版的Redis，但是启动的时候发现数据集合并不完整，需要手动从新导入aop文件**）。
3. Redis 还可以同时使用 AOF 持久化和 RDB 持久化。 在这种情况下， 当 Redis 重启时， 它会优先使用 AOF 文件来还原数据集， 因为 AOF 文件保存的数据集通常比 RDB 文件所保存的数据集更完整。
4. 至可以关闭持久化功能，让数据只在服务器运行时存在。

---
## RDB持久化优点和缺点
### 优点
- RDB是一个非常简洁的文件，它保存了Rdis在某个时间点上的数据集。我觉得这种方式适合大多数中小公司在redis中大多数存储的是热点数据，因为我可以备份最近的24小时内，每个小时备份一次RDB文件，或者是每个月才备份一次。这样可以同时让数据回滚到想要的状态。
- 适合作为灾备恢复，可以把备份好的RDB文件存储到另外一个地方，在需要的时候用回来就可以了。==（试过开发环境的Redis所有数据删除了，因为代码上的问题，导致整个程序都无法运行）==
- RDB 可以最大化 Redis 的性能：父进程在保存 RDB 文件时唯一要做的就是 fork 出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无须执行任何磁盘 I/O 操作。
- RDB在数据比较大的时候会比AOP恢复方式要快

### 缺点
- 当对数据实时性比较高的时候，并不适合使用RDB方式，虽然Redis允许你设置不同的保存点（save point）来控制保存RDB文件的频率，但是，因为RDB文件需要保存整个数据集的状态，所以它并不是一个轻松的操作。 因此你可能会至少 5 分钟才保存一次RDB文件。 在这种情况下， 一旦发生故障停机， 你就可能会丢失好几分钟的数据。
- 每次保存 RDB 的时候，Redis都要fork()出一个子进程，并由子进程来进行实际的持久化工作。 在数据集比较庞大时，fork()可能会非常耗时，造成服务器在某某毫秒内停止处理客户端； 如果数据集非常巨大，并且CPU时间非常紧张的话，那么这种停止时间甚至可能会长达整整一秒。 虽然AOF重写也需要进行fork()，但无论AOF重写的执行间隔有多长，数据的耐久性都不会有任何损失。

### RDB快照设置
1. 默认情况下，Redis将数据集合快照保存为 dump.rdb 的二进制文件中。
2. 可以对Redis进行设置，让它在“N秒内数据集中至少有M个改动”这个条件被满足时，自动保存一次数据
3. 通过调用save或者bgsave ,手动让Redis进行数据保存
> 以下设置会让 Redis 在满足“ 60 秒内有至少有 1000 个键被改动”这一条件时， 自动保存一次数据集

```
save 60 1000
```

#### 快照的运作方式
当 Redis 需要保存 dump.rdb 文件时， 服务器执行以下操作：

- Redis 调用 fork() ，同时拥有父进程和子进程。
- 子进程将数据集写入到一个临时 RDB 文件中。
- 当子进程完成对新 RDB 文件的写入时，Redis 用新 RDB 文件替换原来的 RDB文件，并删除旧的 RDB 文件。
- 这种工作方式使得 Redis 可以从写时复制（copy-on-write）机制中获益

#### RDB文件恢复
- 我的redis启动服务的目录是 /usr/local/bin 下面
- 我启动redis的目录是/root 下面，然后生成的的dump.rdb 文件也是在/root 目录下，假如redis服务器出现问题，挂掉了。那么想要根据rdb恢复数据 
- 将备份文件 (dump.rdb) 移动到 redis 安装目录并启动服务
- 当前目录启动
- ==如果我的dump.rdb 在/root下面，而我到/usr/local/bin这个目录下去启动了redis，那么数据是无法恢复的。只能从 /root 下面启动才能看到之前保存的数据。==

参考链接：
- [http://blog.csdn.net/u010648555/article/details/73433717]

---
### AOF优点
- 使用 AOF 持久化会让 Redis 变得非常耐久（much more durable）：你可以设置不同的 fsync 策略，比如无 fsync ，每秒钟一次 fsync ，或者每次执行写入命令时 fsync 。 AOF 的默认策略为每秒钟 fsync 一次，在这种配置下，Redis 仍然可以保持良好的性能，并且就算发生故障停机，也最多只会丢失一秒钟的数据（ fsync 会在后台线程执行，所以主线程可以继续努力地处理命令请求）
- AOF 文件是一个只进行追加操作的日志文件（append only log）， 因此对 AOF 文件的写入不需要进行 seek ， 即使日志因为某些原因而包含了未写入完整的命令（比如写入时磁盘已满，写入中途停机，等等）， redis-check-aof 工具也可以轻易地修复这种问题。
- Redis 可以在 AOF 文件体积变得过大时，自动地在后台对 AOF 进行重写： 重写后的新 AOF 文件包含了恢复当前数据集所需的最小命令集合。 整个重写操作是绝对安全的，因为 Redis 在创建新 AOF 文件的过程中，会继续将命令追加到现有的 AOF 文件里面，即使重写过程中发生停机，现有的 AOF 文件也不会丢失。 而一旦新 AOF 文件创建完毕，Redis 就会从旧 AOF 文件切换到新 AOF 文件，并开始对新 AOF 文件进行追加操作。
- AOF 文件有序地保存了对数据库执行的所有写入操作， 这些写入操作以 Redis 协议的格式保存， 因此 AOF 文件的内容非常容易被人读懂， 对文件进行分析（parse）也很轻松。 导出（export） AOF 文件也非常简单： 举个例子， 如果你不小心执行了 FLUSHALL 命令， 但只要 AOF 文件未被重写， 那么只要停止服务器， 移除 AOF 文件末尾的 FLUSHALL 命令， 并重启 Redis ， 就可以将数据集恢复到 FLUSHALL 执行之前的状态。

### AOF缺点
- 对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积
- 根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB 。 在一般情况下， 每秒 fsync 的性能依然非常高， 而关闭 fsync 可以让 AOF 的速度和 RDB 一样快， 即使在高负荷之下也是如此。 不过在处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间（latency）。

#### AOF(append-only file)
可以通过修改配置文件来打开 AOF 功能：
```
appendonly yes
```
从现在开始， 每当 Redis 执行一个改变数据集的命令时（比如 SET）， 这个命令就会被追加到 AOF 文件的末尾。

这样的话， 当 Redis 重新启时， 程序就可以通过重新执行 AOF 文件中的命令来达到重建数据集的目的。

#### AOF重写
- 因为 AOF 的运作方式是不断地将命令追加到文件的末尾， 所以随着写入命令的不断增加， AOF 文件的体积也会变得越来越大。
> 举个例子， 如果你对一个计数器调用了 100 次 INCR ， 那么仅仅是为了保存这个计数器的当前值， AOF 文件就需要使用 100条记录）。
> 然而在实际上， 只使用一条 SET 命令已经足以保存计数器的当前值了， 其余 99 条记录实际上都是多余的。
- 为了处理这种情况， Redis 支持一种有趣的特性： 可以在不打断服务客户端的情况下， 对 AOF 文件进行重建（rebuild）。
- 执行 ==BGREWRITEAOF== 命令， Redis 将生成一个新的 AOF 文件， 这个文件包含重建当前数据集所需的最少命令。(Redis 2.4 则可以自动触发 AOF 重写)
 
#### AOF多久执行一次
- 每次有新命令追加到 AOF 文件时就执行一次 fsync ：非常慢，也非常安全。
- 每秒 fsync 一次：足够快（和使用 RDB 持久化差不多），并且在故障时只会丢失 1 秒钟的数据。
- 从不 fsync ：将数据交给操作系统来处理。更快，也更不安全的选择。
- 推荐（并且也是默认）的措施为每秒 fsync 一次， 这种 fsync 策略可以兼顾速度和安全性。
- 总是 fsync 的策略在实际使用中非常慢


#### AOP文件手动导入方式
- 当发现重启redis后数据集出现确实
- 同时本身有开启AOF持久化方式，可以尝试手动导入对应文件
- 这个动作会重新对redis当前的集合进行一次重写，同时这些记录同时都会记录到aof文件中，可以通过重写机制处理

```
redis-cli -h ip -p 6379 -a pass --pipe < appendonly.aof
```
- 如果AOF文件出错了，大多数是由于在对aof重写时停机导致出错，再重启时会载入错误，可以通过部署时间
>为现有的 AOF 文件创建一个备份。
>使用 Redis 附带的 redis-check-aof 程序，对原来的 AOF 文件进行修复。
```
$ redis-check-aof --fix
```
>重启 Redis 服务器，等待服务器载入修复后的 AOF 文件，并进行数据恢复。

---
## 如何选择使用哪一种持久化方式
- 对数据有很强的实时性，并且不能丢失任何数据，可以同时采用两种持久化方式
- 如果你非常关心你的数据， 但仍然可以承受数分钟以内的数据丢失，那么你可以只使用 RDB 持久化。
- 并不建议只采用aof持久化方式，可以定时生成RDB快照方式便于进行数据库备份，确保整个机器出问题时候可以有还原的数据。并且RDB恢复数据比AOF更加快，