# PostgreSQL

## 底层原理

### 表空间逻辑结构

PostgreSQL的存储管理器采用与操作系统类似的分页存储管理方式，即数据在内存中是以页面块的形式存在。每个表文件由多个文件块（文件大小可配置，默认是8KB）组成，每个文件块又可以包含多个元组（tuple），如下图所示。表文件以文件块为单位读入内存中，每一个文件块在内存中形成一个页面块（页面块是文件块在内存中的存在形式，二者大小相同，很多时候不区分这两个概念）。同样，文件的写入也是以页面块为单位。PostgreSQL是传统的行式数据库，是以元组为单位进行数据存储的。一个文件块中存放多个完整的元组。

![file_block](https://niyanchun.com/usr/uploads/2016/06/file_block.png)

### 文件块物理结构

每个堆文件由多个文件块组成，在物理磁盘中的存储形式如下图：

![file_block2](https://niyanchun.com/usr/uploads/2016/06/file_block2.png)





#### Tuple的增删改

![img](https://upload-images.jianshu.io/upload_images/2960526-a7696d30c717ab4f.png?imageMogr2/auto-orient/strip|imageView2/2/w/775/format/webp)

##### 插入

在执行插入操作时，PostgreSQL 会直接将某个新的 Tuple 插入到目标表的某个页中：



![image](https://user-gold-cdn.xitu.io/2019/1/31/168a3ebb63373f8d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



假如某个 txid 为 99 的事务插入了新的 Tuple，那么该 Tuple 的头域会被设置为如下值：

* t_xmin 与创建该 Tuple 的事务的 txid 保持一致，即 99
* t_xmax 被设置为 0，因为其还未被删除或者更新
* t_cid 被设置为 0，因为该 Tuple 是由事务中的首个 Insert 命令创建的
* t_ctid 被设置为了 `(0, 1)`，即指向了自己

##### 删除

在删除操作中，目标 Tuple 会被先逻辑删除，即将 t_xmax 的值设置为当前删除该 Tuple 的事务的 txid 值。



![image](https://user-gold-cdn.xitu.io/2019/1/31/168a3ebbc74af31b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



当该事务被提交之后，PostgreSQL 会将该 Tuple 标记为 Dead Tuple，并随后在 VACUUM 处理过程中被彻底清除。

##### 更新

在更新操作时，PostgreSQL 会首先逻辑删除最新的 Tuple，然后插入新的 Tuple：



![image](https://user-gold-cdn.xitu.io/2019/1/31/168a3ebb62dd766c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



上图所示的行被 txid 为 99 的事务插入，被 txid 为 100 的事务连续更新两次；在该事务提交之后，Tuple_2 与 Tuple_3 就会被标记为 Dead Tuples。

## 应用

### MVCC

在并发操作中，当正在写时，如果用户在读，可能会产生数据不一致的问题。比如一行的前半部分刚写入，后半部分还没有写入，这时读的人可能读取到的数据行，其前半部分数据是新的，后半部分数据是旧的，这样就导致了数据一致性问题。解决这个问题最简单的办法是使用读写锁。写的时候，不允许读，正在读的时候也不允许写，但这种方法导致读和写不能并发。于是，就有了让读写并发的方法（MVCC）。

实现MVCC的方法有两种

1. 写数据时，把旧数据移到一个单独的地方，如回滚段中，其他人读数据时，从回滚段中把旧的数据读出来。
2. 写数据时，旧数据不删除，而是把新数据插入。

而PostgreSQL正是采用了第二种方式。Oracle和MySQL中的innodb引擎使用的是第一种方法。

### 事务

#### 事务ID

当某个事务开启时，PostgreSQL会为它分配唯一的 Transaction ID(txid)；txid 是 32 位无类型整型值。

从3开始分配，0-代表无效事务，1-代表初始化，2-代表事务冻结。

##### 事务新旧比较

当两个事物同事访问记录时，通过参考tmin和tmax的标记可判断记录的版本，然后根据版本号与自己当前的事务标识进行比较，确定自己的数据权限。 

判断公式：（（int32）id1 -id2） < 0  

##### 事务回收

满足一定条件时，将其冻结，事务ID标记为2，从而实现了将其对应的XID回收的操作。

#### 快照

事务快照是用来存储数据库的事务运行情况。一个事务快照的创建过程可以概括为：

* 查看当前所有的未提交并活跃的事务，存储在数组中
* 选取未提交并活跃的事务中最小的XID，记录在快照的xmin中
* 选取所有已提交事务中最大的XID，加1后记录在xmax中

例如如下场景：

#### ![image](http://www.postgres.cn/images/news/2019/20190122_0943_03.png)

最后已结束事务为200，最早仍活跃事务为125，则：

```
xmax = 200 + 1 = 201
xmin = 125
```

#### 可见性

其中根据xmin和xmax的定义，事务和快照的可见性可以概括为：

* XID ∈ [1,xmin),过去的事务,对此快照均可见; 
* XID ∈ [xmin,xmax),考虑事务的情况,仍处于IN_PROGRESS状态的,不可见;COMMITED状态,可见;ABORTED状态,不可见; 
* XID ∈ [xmax,∞),未来的事务,对此快照均不可见;

#### 快照隔离

简单的理解就是，写一个值时，不要在这个值上面做修改，而是创建一个新的版本，也就是所谓快照。而读一个值会读取最近提交成功了的那个版本的数据。读读肯定是没问题的，读写也没问题，读到的是最后提交的版本。至于写写，如果在提交的时候检测到有冲突，将这个事务abort掉就行了。所有的操作完全不阻塞，相比加锁等待的方案，性能一下子就上去了。

存在写偏离（write skew）问题

```
数据库约束： A1+A2>0
A1，A2 实际值都为100
事务T1：
If （read （A1） +read （A2） >= 200）
{
Set A1 = A1 – 200
}
事务T2：
If （read （A1） +read （A2） >= 200）
{
Set A2 = A2 – 200
}
```

事务T2 与事务T1 并发执行相同的语句，两个事务都会执行，执行成功后A1= -100 ，A2= -100 ， A1+A2=-200，显然违背完整性约束。

#### 回滚

事务回滚时，产生的数据不会立即清除。因为数据可能刷新到磁盘，再删除，会浪费IO。PG是通过维护事务的状态来确定可见性。

优点：事务回滚可以立即完成，无论事务有多少操作；数据可以很多次更新，不必担心回滚段用完。

缺点：旧数据需要被清理；旧数据的存在会导致查询更慢。

#### DDL事务 

大多数DDL可以包含在一个事务中，且可回滚。

#### SavePoint

可以回滚到指定保存点，而不必全部回滚

#### 事务隔离级别

##### PG的事务

| 隔离级别 | 脏读               | 不可重复读 | 幻读               | 序列化异常 |
| -------- | ------------------ | ---------- | ------------------ | ---------- |
| 读未提交 | 允许，但不在 PG 中 | 可能       | 可能               | 可能       |
| 读已提交 | 不可能             | 可能       | 可能               | 可能       |
| 可重复读 | 不可能             | 不可能     | 允许，但不在 PG 中 | 可能       |
| 可序列化 | 不可能             | 不可能     | 不可能             | 不可能     |

PostgreSQL中：

* 默认隔离级别是**读已提交**（read committed）；
* 可以请求四中级别的任意一种，但对于内部其实只有读已提交（看到的是**当前查询开始时的快照**）、可重复读（看到的是**当前事务开始时的快照**）、可串行化三种级别；
* 选择读未提交时，实际上用的是读已提交；
* 选择重复读时，不会发生幻读；

##### PG串行化的实现-可串行化快照隔离

可串行化快照隔离(serializable snapshot isolation或SSI)是在快照隔离级别之上，支持串行化。

快照隔离存在的write skew的问题，本质上需要至少两个条件：

* 有读写冲突
* 依赖成环

如果可以破坏这些条件，就可以避免write skew，达到可串行化了。所以要做的就是检测不同事务之间的读写依赖是否形成环。

在PostgreSQL中，使用以下方法来实现SSI：

1. 利用**SIREAD LOCK(谓词锁)**记录每一个事务访问的对象(tuple、page和relation)；
2. 在事务写堆表或者索引元组时利用SIREAD LOCK监测是否存在冲突；
3. 如果发现到冲突(即序列化异常)，abort该事务。

###### 谓词锁

类似于排它锁，但不属于特定的对象（例如，表中的一行），它属于所有符合某些搜索条件的对象。



## 并行查询

### 原理

![img](https://ask.qcloudimg.com/http-save/yehe-1173439/z4m8j6dtq7.jpeg?imageView2/2/w/1620)

### 执行情况

```
Gather  (cost=1000.55..69893.93 rows=199 width=795) (actual time=611.144..648.942 rows=800 loops=1)
  Workers Planned: 4
  Workers Launched: 4
  ->  Nested Loop  (cost=0.55..68874.03 rows=50 width=795) (actual time=419.869..423.704 rows=160 loops=5)
        ->  Parallel Seq Scan on netdisk_file  (cost=0.00..68445.41 rows=50 width=220) (actual time=419.799..419.914 rows=162 loops=5)
              Filter: ((file_path)::text ~~ '/file/yiDaPXZLHe/LA21jCpupj/WNguI/RVrAe/LInCZ/%'::text)
              Rows Removed by Filter: 402039
        ->  Index Scan using file_meta_data_pkey on netdisk_file_meta_data  (cost=0.55..8.57 rows=1 width=575) (actual time=0.023..0.023 rows=1 loops=808)
              Index Cond: ((id)::text = (netdisk_file.file_meta_data_id)::text)
              Filter: (size > 100)
Planning Time: 0.219 ms
Execution Time: 649.016 ms
```

### 性能

基于9.6

![pic1](https://raw.githubusercontent.com/digoal/blog/master/201610/20161001_01_pic_001.png)

![pic2](https://github.com/digoal/blog/blob/master/201610/20161001_01_pic_002.png?raw=true)



## sql高级特性

### with查询

with查询是在复杂查询中定义一个辅助语句（可理解为一个临时表）。这种特性称为CTE（common table expressions）

#### 复杂查询

减少嵌套以简化SQL,提高可读性。

```sql
WITH regional_sales AS (
    SELECT region, SUM(amount) AS total_sales
    FROM orders
    GROUP BY region
), top_regions AS (
    SELECT region
    FROM regional_sales
    WHERE total_sales > (SELECT SUM(total_sales)/10 FROM regional_sales)
)
SELECT region,
       product,
       SUM(quantity) AS product_units,
       SUM(amount) AS product_sales
FROM orders
WHERE region IN (SELECT region FROM top_regions)
GROUP BY region, product;
```

它只显示在高销售区域每种产品的销售总额。`WITH`子句定义了两个辅助语句`regional_sales`和`top_regions`，其中`regional_sales`的输出用在`top_regions`中而`top_regions`的输出用在主`SELECT`查询。这个例子可以不用`WITH`来书写，但是我们必须要用两层嵌套的子`SELECT`。使用这种方法要更简单些。

#### 递归查询

```sql
WITH RECURSIVE r AS (
SELECT * FROM netdisk_file WHERE netdisk_file."id" = '7f6d16ca7dc7453984d46fb2e564fffe' 
UNION ALL
SELECT netdisk_file.* FROM netdisk_file,r WHERE r.parent_file_id = netdisk_file.id 
) 
SELECT * FROM r
```



![image](https://github.com/fd159951/imageRepository/blob/master/1568704255514.png?raw=true)

![image](https://github.com/fd159951/imageRepository/blob/master/1568704377547.png?raw=true)

### retruning

返回插入、更新、删除后的数据

![image](https://github.com/fd159951/imageRepository/blob/master/1568705090820.png?raw=true)

### UPSERT

当插入遇到约束错误时，直接返回或者改为执行 UPDATE。

```sql
insert into file(id,file_name) values (1,'test') on conflict (id) 
do update set file_name=excluded.file_name
insert into file(id,file_name) values (1,'test') on conflict (id) do nothing
```

### 聚合函数

对结果集计算，返回一行

#### array_agg函数

能将某个字段的所有行连接成数组。

例如，在下city表中：

| country | city |
| ------- | ---- |
| 中国    | 深圳 |
| 中国    | 香港 |
| 日本    | 东京 |
| 日本    | 大阪 |

执行：

```sql
select country，array_agg(city) from city group by country
```

結果：

| country | array_agg    |
| ------- | ------------ |
| 日本    | {东京，大阪} |
| 中国    | {深圳，香港} |



### 窗口函数

对结果集计算，返回多行

准备数据：

| subject | stu_name | score |
| ------- | -------- | ----- |
| Chinese | francs   | 70    |
| Chinese | matiler  | 70    |
| Chinese | tutu     | 80    |
| English | matiler  | 75    |
| English | francs   | 90    |
| English | tutu     | 60    |



#### avg()  OVER()

聚合函数后接OVER属性的窗口函数表示在一个查询结果集上应用聚合函数。

```sql
/* 统计每门课的平均分 */
SELECT  subject , stu_name , score ,  avg( score ) OVER(PARTITION  BY  s ubject )  FROM  score
```

| subject | stu_name | score | avg  |
| ------- | -------- | ----- | ---- |
| Chinese | francs   | 70    | 73.3 |
| Chinese | matiler  | 70    | 73.3 |
| Chinese | tutu     | 80    | 73.3 |
| English | matiler  | 75    | 75   |
| English | francs   | 90    | 75   |
| English | tutu     | 60    | 75   |

#### row_number()

对结果集分组后的数据标注行号

```sql
/* 统计每门课的排名 */
select *,row_number() over(partition by subject) from score
```

| subject | stu_name | score | rownum |
| ------- | -------- | ----- | ------ |
| Chinese | francs   | 70    | 3      |
| Chinese | matiler  | 70    | 2      |
| Chinese | tutu     | 80    | 1      |
| English | matiler  | 75    | 2      |
| English | francs   | 90    | 1      |
| English | tutu     | 60    | 3      |

#### first_value  ()

取出结果集每个分组的第一行数据的值

```sql
/* 取出每门课的最高分 */
select first_value(score)  OVER(  PARTITION  BY  subject ORDER  BY  scoredesc),* FROM  score
```

| subject | stu_name | score | first_value |
| ------- | -------- | ----- | ----------- |
| Chinese | francs   | 70    | 80          |
| Chinese | matiler  | 70    | 80          |
| Chinese | tutu     | 80    | 80          |
| English | matiler  | 75    | 90          |
| English | francs   | 90    | 90          |
| English | tutu     | 60    | 90          |
