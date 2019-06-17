## SQL Server 的查询过程、执行计划

### Building Blocks的概念

SQL Server的每一个查询都是由Building Block组成的集合，Building Block分为两种，operators和iterators。一个iterator从它的子iterator中获取数据，经过处理后返回给它的父iterator。

所有iterator都实现了一个接口， 这个接口中有两个函数，Open和GetRow。Open函数告诉operator准备输出结果行，GetRow函数返回一个新的结果。

Query Plan是一个由iterators组成的树形结构，控制流从根到叶子流动，数据流从叶子到根流动。

### Iterators的属性

- 内存：iterator的执行需要申请内存，当执行一个iterator之前，会估计它占用的内存并预留出足够的内存。
- 非阻塞iterator和阻塞iterator：非阻塞iterator从子iterator消费输入行，和向父iterator生产输出行是同时进行的，接收到一行输入就立刻处理和输出。阻塞iterator得到所有的子iterator输入行并处理结束后，才会向父iterator输出。非阻塞iterator更适用于OLTP场景。
- 动态游标支持：动态游标查询有一些特别的属性，如果一个Query Plan是一个动态游标Query Plan，那么它必须可以一次返回一个集合，必须可以正向Scan和反向Scan，必须可以获取Scroll Locks。支持动态游标的Query Plan必须满足，组成它的每个iterator必须能够重置状态、正反向Scan、非阻塞。所以不是每个Query Plan都能支持动态游标。

### Scan 和 Seek

- Scan ： 扫描整张表，通常是由于要查询的列不存在索引。如下

    `|--Table Scan(OBJECT:([ORDERS]), WHERE:([ORDERKEY]=(2)))`
    
- Seek： 查询的列存在索引，一般会采用Seek查询索引列，当然也有特殊，要考虑占用Memory和IO次数。
    
    `|--Index Seek(OBJECT:([ORDERS].[OKEY_IDX]), SEEK:([ORDERKEY]=(2)) ORDERED FORWARD)`


 -  | Scan | Seek 
---|---|---
堆 | Table Scan | -
聚集索引 | Clustered Index Scan | Clustered Index Seek
非聚集索引 | index Scan | Index Seek

### Book Lookup

- 原因：where从句中的谓词是非聚集索引，不包含需要查询的列，需要先seek非聚集索引，再从seek行的聚集索引查找到查询列。

```sql
create table T (a int, b int, c int);
create unique clustered index T_clu_a on T(a);
create index T_b on T(b);

select c from T where b = 2;
```

这里的查询需要首先利用非聚集索引b，seek到相应的行，因为b是非聚集索引，索引b只包含a、b两列，所以需要再根据聚集索引 a 查找 c。

Book Lookup 查询聚集索引是随机I/O，因此很耗时间，因此当查询出现大量Book Lookup时，我们通常是修改查询SQL或者添加其他索引。

### Seek的谓词

#### 单列索引, 假设索引是 a

当索引只有单列时，SQL Plan中下面几种查询谓词可以用到该索引

- a = 3.14
- a > 100
- a between 0 and 99
- a like ‘abc%’
- a in (2, 3, 5, 7)

下面的查询谓词不会用到索引。

- ABS(a) = 1
- a + 1 = 9
- a like ‘%abc’

#### 多列索引，假设索引是(a, b)

当索引有多列时，下面的查询谓词会用到索引Seek。

- a = 3.14 and b = ‘pi’
- a = ‘xyzzy’ and b <= 0

下面的查询谓词不会用到索引。

- b = 0
- a + 1 = 9 and b between 1 and 9
- a like ‘%abc’ and b in (1, 3, 5)

唯一聚集索引(unique clustered index) 包含表所有的列，并且索引的顺序即是数据在磁盘中的物理顺序。

非聚集索引只包含它的索引列和所有的聚集索引列。

### 索引的权衡

同样一个SQL语句，由于查询到的数据行数不同，所执行的SQL Plan可能不同，这是由于数据库的Optimizer对使用内存空间大小和I/O次数的权衡所致。

- 如果每次查询都用聚集索引，那每次查询都要将整行的数据读到内存，而我们需要的往往是其中的某几列，这显然很浪费内存资源。
- 如果总是使用非聚集索引，可能会造成BookMark Lookup的次数过多，导致随机I/O的次数过多。

因此，权衡内存使用和随机I/O而改善SQL执行计划，是任何一款SQL优化器必须要做的。

### Joins，表的连接
- Inner join， 内连接
- Outer join， 外连接
- Cross join， 笛卡尔积
- Cross apply，带参数的动态笛卡尔积(暂时这么解释)
- Semi-join， 半连接
- Anti-semi-join， 反半连接

Semi-join。所谓的semi-join是指semi-join子查询。当一张表在另一张表找到匹配的记录之后，semi-jion返回第一张表中的记录。与条件连接相反，即使在右节点中找到几条匹配的记录，左节点的表也只会返回一条记录。另外，右节点的表一条记录也不会返回。半连接通常使用IN或 EXISTS 作为连接条件。

Anti-semi-join则与semi-join相反，即当在第二张表没有发现匹配记录时，才会返回第一张表里的记录。当使用not exists/not in的时候会用到，两者在处理null值的时候会有所区别。

#### Nested Loops Join

嵌套循环连接，是使用的比较频繁的连接方式，它连接两张表，将其中一张表的每一行和另一张表的每一行根据是否符合连接条件相连接。下面是伪代码。

```python
for each row R1 in the outer table
    for each row R2 in the inner table
        if R1 joins with R2
            return (R1, R2)
```
例子

```sql
select *
from Sales S inner join Customers C
on S.Cust_Id = C.Cust_Id
option(loop join)
```

因为没有任何索引，所以SQL Plan如下

```
|--Nested Loops(Inner Join, WHERE:([C].[Cust_Id]=[S].[Cust_Id]))
   |--Table Scan(OBJECT:([Customers] AS [C]))
   |--Table Scan(OBJECT:([Sales] AS [S]))
```
加一个唯一聚集索引

```sql
create clustered index CI on Sales(Cust_Id)
```
SQL Plan

```
|--Nested Loops(Inner Join, OUTER REFERENCES:([C].[Cust_Id]))
   |--Table Scan(OBJECT:([Customers] AS [C]))
   |--Clustered Index Seek(OBJECT:([Sales].[CI] AS [S]), SEEK:([S].[Cust_Id]=[C].[Cust_Id]) ORDERED FORWARD)
```
Nested loops join支持的连接类型如下。
- Inner join
- Left outer join
- Cross join
- Cross apply and outer apply
- Left semi-join and left anti-semi-join

不支持所有的右连接，试想如果支持右连接，就要返回(null, R2), 每次输入外部表的一行，就要遍历一次内部表，这样就会出现非常多重复的(null, R2)。

#### Merge Join ，合并连接

合并连接伪代码如下。
```python
get first row R1 from input 1
get first row R2 from input 2
while not at the end of either input
    begin
        if R1 joins with R2
            begin
                return (R1, R2)
                get next row R2 from input 2
            end
        else if R1 < R2
            get next row R1 from input 1
        else
            get next row R2 from input 2
    end
```

Merge Join要求两个表必须根据Where后的equal谓词排序，如果谓词是索引的话，SQL Plan中就不用排序了，如果不是索引，则SQL plan中需要排序operator。

- One-to-many merge join

一对多合并连接，要求查询必须有equal谓词，就是where中要有 = 号。外表的排序列(也是equal谓词中的列)必须是unique的，这样才能执行一对多合并连接。

一对多合并连接保证外表的连接列是unique的，所以外表保证了没有重复行，所有内部表是一直往下走的，不会出现回头。不需要一个外部的tempdb保存外表上一行匹配的内表行。

- Many-to-many merge join

多对多合并，可以没有equal谓词，没有equal谓词的时候，相当于Full Outer Join。

多对多的合并连接需要一个外部的tempdb保存上一个外表行匹配的内表行，如果外表的下一行和上一行相同，直接把tempdb中的表与之连接即可。

例子。

```sql
select *
from T1 join T2 on T1.a = T2.a
option (merge join)
```
对应的SQL plan

```
|--Merge Join(Inner Join, MANY-TO-MANY MERGE:([T1].[a])=([T2].[a]), RESIDUAL:([T2].[a]=[T1].[a]))
   |--Sort(ORDER BY:([T1].[a] ASC))
   |    |--Table Scan(OBJECT:([T1]))
   |--Sort(ORDER BY:([T2].[a] ASC))
        |--Table Scan(OBJECT:([T2]))
```

给T1添加一个唯一聚集索引

```sql
create unique clustered index T1ab on T1(a, b)
```

SQL Plan

```
|--Merge Join(Inner Join, MANY-TO-MANY MERGE:([T1].[a])=([T2].[a]), RESIDUAL:([T2].[a]=[T1].[a]))
   |--Clustered Index Scan(OBJECT:([T1].[T1ab]), ORDERED FORWARD)
   |--Sort(ORDER BY:([T2].[a] ASC))
        |--Table Scan(OBJECT:([T2]))
```

给T2添加一个唯一聚集索引

```sql
create unique clustered index T2a on T2(a)
```

SQL Plan

```
|--Merge Join(Inner Join, MERGE:([T2].[a])=([T1].[a]), RESIDUAL:([T2].[a]=[T1].[a]))
   |--Clustered Index Scan(OBJECT:([T2].[T2a]), ORDERED FORWARD)
   |--Clustered Index Scan(OBJECT:([T1].[T1ab]), ORDERED FORWARD)
```

#### Hash Join ，哈希连接

前面说到的Merge Join，必须对连接列进行排序，速度上很慢，Hash Join其实是一种空间换时间的理念，不用进行排序，但是查询中必须要有equal谓词。

伪代码如下。

```python
for each row R1 in the build table
    begin
        calculate hash value on R1 join key(s)
        insert R1 into the appropriate hash bucket
    end
for each row R2 in the probe table
    begin
        calculate hash value on R2 join key(s)
        for each row R1 in the corresponding hash bucket
            if R1 joins with R2
                return (R1, R2)
    end
```

因为hash join要用到一个hash table保存外表的key经过hash之后的列，所以需要很大的内存空间，容易出现内存溢出。在内存溢出时候，数据库内核会将溢出的bucket暂时保存在磁盘中，当内存中hash join执行完之后，再将磁盘中的bucket读到内存中，继续遍历内表进行hash Join。

##### Left deep vs. right deep vs. bushy hash join trees

- Left deep hash join tree

Left deep的方式是本次hashjoin的结果作为下一次hashjoin的bulid表。这种操作的好处是相邻的两个hash join可以同时进行，当第一个hash join 执行到probe操作时，每次在hashtable中匹配一个连接，可以将这个已连接的行输出到下一个hashjoin，下一个hashjoin同时进行build hashtable操作。第三次的hashjoin的build就可以复用第一次hashtable的内存空间，所以left deep比较节省空间。

- right deep hash join tree

right deep的方式是本次hashjoin连接表作为下一次hashjoin的probe表。这种操作的好处是可以并行的进行build  hashtable。因为每次build hashtable是相互隔离的。但是它不能共用内存空间了。

- bushy hash join tree

同时进行多个hashjoin，然后将这几个hashjoin的结果分别作为build表和probe表。这种方式的并行性很强，速度快。

要根据数据行的大小和具体情况选择是利用left deep还是right deep还是 bushy。

hash join的例子

```sql
select *
from T1 join T2 on T1.a = T2.a 
```

SQL Plan
```
|--Hash Match(Inner Join, HASH:([T1].[a])=([T2].[a]), RESIDUAL:([T2].[a]=[T1].[a]))
   |--Table Scan(OBJECT:([T1]))
       |--Table Scan(OBJECT:([T2]))
```

### CASE表达式子查询

示例
```sql
create table T1 (a int, b int, c int);
select
    case
        when T1.a > 0 then
            T1.b
        else
            T1.c
    end
from T1;
```
SQL Plan

```
|--Compute Scalar(DEFINE:([Expr1004]=CASE WHEN [T1].[a]>(0) THEN [T1].[b] ELSE [T1]. END))
   |--Table Scan(OBJECT:([T1]))
```

Expr1004被赋值为CASE表达式的值，Expr1004可以被后面执行的operator访问到。

#### 带有when子查询的CASE

```sql
create table T2 (a int, b int)
select
    case
        when exists (select * from T2 where T2.a = T1.a) then
            T1.b
        else
            T1.c
    end
from T1
```

对应的SQL Plan如下

```
|--Compute Scalar(DEFINE:([Expr1009]=CASE WHEN [Expr1010] THEN [T1].[b] ELSE [T1]. END))
   |--Nested Loops(Left Semi Join, OUTER REFERENCES:([T1].[a]), DEFINE:([Expr1010] = [PROBE VALUE]))
        |--Table Scan(OBJECT:([T1]))
        |--Table Scan(OBJECT:([T2]), WHERE:([T2].[a]=[T1].[a]))
```
在Left Semi Join中可以看到一个PROBE，探针，表示是否匹配，如果探针的值是true，则Nested loops会立即返回。

#### THEN 子查询

创建一个新的表T3.

```sql
create table T3 (a int unique clustered, b int);
insert T1 values(0, 0, 0);
insert T1 values(1, 1, 1);
select
    case
        when T1.a > 0 then
            (select T3.b from T3 where T3.a = T1.b)
        else
            T1.c
    end
from T1;
```
它对应的SQL Plan是

```
|--Compute Scalar(DEFINE:([Expr1008]=CASE WHEN [T1].[a]>(0) THEN [T3].[b] ELSE [T1]. END))
   |--Nested Loops(Left Outer Join, PASSTHRU:(IsFalseOrNull [T1].[a]>(0)), OUTER REFERENCES:([T1].[b]))
        |--Table Scan(OBJECT:([T1]))
        |--Clustered Index Seek(OBJECT:([T3].[UQ__T3__412EB0B6]), SEEK:([T3].[a]=[T1].[b]) ORDERED FORWARD)
```

这里出现了一个新的专有名词，PASSTHRE。首先对T1进行Table Scan，接着PASSTHRU发挥作用，如果PASSTHRU是true，即isFalseOrNull [T1].[a] > (0)是true，则T1.a <= 0或T1.a = null, 这个时候就不再对T3进行Clustered Index Seek了直接将T1返回给上层。如果PASSTHRU是false，则继续SeekT3，也就是执行then中的语句。符合原SQL语句的逻辑。

#### else子查询和多个When子查询的情况

```sql
create table T4 (a int unique clustered, b int)

create table T5 (a int unique clustered, b int)

select
    case
        when T1.a > 0 then
            (select T3.b from T3 where T3.a = T1.a)
        when T1.b > 0 then
            (select T4.b from T4 where T4.a = T1.b)
        else
            (select T5.b from T5 where T5.a = T1.c)
    end
from T1
```
对应的SQL Plan
```
|--Compute Scalar(DEFINE:([Expr1016]=CASE WHEN [T1].[a]>(0) THEN [T3].[b] ELSE CASE WHEN [T1].[b]>(0) THEN [T4].[b] ELSE [T5].[b] END END))
       |--Nested Loops(Left Outer Join, PASSTHRU:([T1].[a]>(0) OR [T1].[b]>(0)), OUTER REFERENCES:([T1].))
            |--Nested Loops(Left Outer Join, PASSTHRU:([T1].[a]>(0) OR IsFalseOrNull [T1].[b]>(0)), OUTER REFERENCES:([T1].[b]))
            |    |--Nested Loops(Left Outer Join, PASSTHRU:(IsFalseOrNull [T1].[a]>(0)), OUTER REFERENCES:([T1].[a]))
            |    |    |--Table Scan(OBJECT:([T1]))
            |    |    |--Clustered Index Seek(OBJECT:([T3].[UQ__T3__164452B1]), SEEK:([T3].[a]=[T1].[a]) ORDERED FORWARD)
            |    |--Clustered Index Seek(OBJECT:([T4].[UQ__T4__182C9B23]), SEEK:([T4].[a]=[T1].[b]) ORDERED FORWARD)
            |--Clustered Index Seek(OBJECT:([T5].[UQ__T5__1A14E395]), SEEK:([T5].[a]=[T1].) ORDERED FORWARD)
```
由多个PASSTHRU控制是否执行对应的then子查询。很好理解。

### ANDs和ORs子查询

#### AND子查询

```sql
select * from T1
where exists (select * from T2 where T2.a = T1.a) and
not exists (select * from T3 where T3.a = T1.b);
```

它对应的SQL Plan是
```
|--Nested Loops(Left Anti Semi Join, WHERE:([T3].[a]=[T1].[b]))
   |--Nested Loops(Left Semi Join, WHERE:([T2].[a]=[T1].[a]))
   |    |--Table Scan(OBJECT:([T1]))
   |    |--Table Scan(OBJECT:([T2]))
   |--Table Scan(OBJECT:([T3]))
```

SQL Server先执行exsits子查询，然后以这个结果为左表连接not exists执行的结果。

#### OR 子查询

```sql
select * from T1
where exists (select * from T2 where T2.a = T1.a) or
      exists (select * from T3 where T3.a = T1.b)
```


这个时候不能直接连接了，因为是Or。通常Optimizer会把这样的SQL进行转换。上面的SQL相当于下面的。

- 第一种方式

```sql
select * from T1

where exists
    (
    select * from T2 where T2.a = T1.a
    union all
    select * from T3 where T3.a = T1.b
    )
```
SQL Plan

```
 |--Nested Loops(Left Semi Join, OUTER REFERENCES:([T1].[a], [T1].[b]))
       |--Table Scan(OBJECT:([T1]))
       |--Concatenation
            |--Table Scan(OBJECT:([T2]), WHERE:([T2].[a]=[T1].[a]))
            |--Table Scan(OBJECT:([T3]), WHERE:([T3].[a]=[T1].[b]))
```

Optimizer巧妙地将查询变成了只有一个exists子查询的SQL，子查询是UNION操作。

- 第二种方式

```sql
select * from T1
where exists (select * from T2 where T2.a = T1.a)
union
select * from T1
where exists (select * from T3 where T3.a = T1.b)
```
SQL Plan

```
 |--Sort(DISTINCT ORDER BY:([Bmk1000] ASC))
       |--Concatenation
            |--Hash Match(Right Semi Join, HASH:([T2].[a])=([T1].[a]), RESIDUAL:([T2].[a]=[T1].[a]))
            |    |--Table Scan(OBJECT:([T2]))
            |    |--Table Scan(OBJECT:([T1]))
            |--Hash Match(Right Semi Join, HASH:([T3].[a])=([T1].[b]), RESIDUAL:([T3].[a]=[T1].[b]))
                 |--Table Scan(OBJECT:([T3]))
                 |--Table Scan(OBJECT:([T1]))
```

注意这个时候最后由一个SORT DISTINCT操作，因为现在将原SQL分为两个查询的UNION，出现了重复的行，所以SORT DISTINCT操作是为了去重。

### 聚合

聚合是指通过对查询结果的一系列计算(聚合函数)，返回给用户一行结果的操作。将多行通过分类计算返回少量行。

#### 标量聚合
```sql
create table t (a int, b int, c int)
select count(*) from t;
```

对应的SQL Plan
```
 |--Compute Scalar(DEFINE:([Expr1004]=CONVERT_IMPLICIT(int,[Expr1005],0)))
       |--Stream Aggregate(DEFINE:([Expr1005]=Count(*)))
            |--Table Scan(OBJECT:([t]))
```
这里的Conpute Scalar的作用是将计算的到的Bigint类型转化成int。

```sql
select min(a), max(b) from t
```

SQL Plan
```
 |--Stream Aggregate(DEFINE:([Expr1004]=MIN([t].[a]), [Expr1005]=MAX([t].[b])))
       |--Table Scan(OBJECT:([t]))
```
#### 标量Distinct

```sql
select count(distinct a) from t
```

SQL Plan
```
|--Compute Scalar(DEFINE:([Expr1004]=CONVERT_IMPLICIT(int,[Expr1007],0)))
   |--Stream Aggregate(DEFINE:([Expr1007]=COUNT([t].[a])))
        |--Sort(DISTINCT ORDER BY:([t].[a] ASC))
             |--Table Scan(OBJECT:([t]))
```
这里需要Distinct，再执行count，用到了Order By进行排序，然后从上到下Table Scan计算a的不重复个数。

#### 多个Distinct

```sql
select count(distinct a), count(distinct b) from t
```

SQL Plan
```
|--Nested Loops(Inner Join)
   |--Compute Scalar(DEFINE:([Expr1004]=CONVERT_IMPLICIT(int,[Expr1010],0)))
   |    |--Stream Aggregate(DEFINE:([Expr1010]=COUNT([t].[a])))
   |         |--Sort(DISTINCT ORDER BY:([t].[a] ASC))
   |              |--Table Scan(OBJECT:([t]))
   |--Compute Scalar(DEFINE:([Expr1005]=CONVERT_IMPLICIT(int,[Expr1011],0)))
        |--Stream Aggregate(DEFINE:([Expr1011]=COUNT([t].[b])))
             |--Sort(DISTINCT ORDER BY:([t].[b] ASC))
                  |--Table Scan(OBJECT:([t]))
```
可以发现当多个Distinct计数时，是逐个查询每两个Distinct，然后做一个inner join，结果作为下一个Distinct计数查询的输入。

#### Hash 聚合

Hash聚合一般用在有Group By子句的SQL Plan中。是一种以空间换取时间的做法。

伪代码如下。

```python
for each input row
  begin
    calculate hash value on group by column(s)
    check for a matching row in the hash table
    if we do not find a match
      insert a new row into the hash table
    else
      update the matching row with the input row
  end
output all rows in the hash table
```

其实和hash join有点像，只不过这里的hash是以group by从句中的列作为key的，同一个bucket中的行执行聚合。

当然这种聚合方式也会出现内存溢出的情况，处理方式和hash join基本一致。

```sql
create table t (a int, b int, c int)

set nocount on
declare @i int
set @i = 0
while @i < 100
  begin
    insert t values (@i % 10, @i, @i * 3)
    set @i = @i + 1
  end
  
select sum(b) from t group by a
```
这里插入了100条数据。SQL Plan如下

```
|--Compute Scalar(DEFINE:([Expr1004]=CASE WHEN [Expr1010]=(0) THEN NULL ELSE [Expr1011] END))
   |--Stream Aggregate(GROUP BY:([t].[a]) DEFINE:([Expr1010]=COUNT_BIG([t].[b]), [Expr1011]=SUM([t].[b])))
        |--Sort(ORDER BY:([t].[a] ASC))
             |--Table Scan(OBJECT:([t]))
```

是没有使用hash 聚合的，因为100条数据太少了，优化器认为，对100条数据进行排序的花费，和其hash聚合对内存和时间的花费差别不大，所以不用采用复杂的hash聚合。

下一条SQL

```sql
truncate table t

declare @i int
set @i = 100

while @i < 1000
  begin
    insert t values (@i % 100, @i, @i * 3)
    set @i = @i + 1
  end
  
select sum(b) from t group by a
```

这里插入了1000条数据，SQL Plan如下。
```
 |--Compute Scalar(DEFINE:([Expr1004]=CASE WHEN [Expr1010]=(0) THEN NULL ELSE [Expr1011] END))
       |--Hash Match(Aggregate, HASH:([t].[a]), RESIDUAL:([t].[a] = [t].[a]) DEFINE:([Expr1010]=COUNT_BIG([t].[b]), [Expr1011]=SUM([t].[b])))
            |--Table Scan(OBJECT:([t]))
```
这里用到了hash聚合，优化器任务，排序1000条数据很慢，而1000条数据占用的内存不大，所以选择Hash聚合。

#### Hash聚合实现Distinct

```sql
select distinct a from t
```

对应的SQL Plan如下。
```
 |--Hash Match(Aggregate, HASH:([t].[a]), RESIDUAL:([t].[a] = [t].[a]))
       |--Table Scan(OBJECT:([t]))
```
这里使用聚合的RESIDUAL是t.a = t.a，很明显聚合后的结果是，有多少中a，就有多少个hash bucket，这样就能做到去重。

### 子查询的去相关(Decorrelating)

当子查询和父查询没有相关性的时候，我们就可以并行地执行一个SQL语句。所以，去相关性是SQL优化的一个重要方法。

下面一个例子.
```sql
create table T1 (a int, b int)
create table T2 (a int, b int, c int)

select *
from T1
where T1.a in (select T2.a from T2 where T2.b = T1.b)
```

SQL Plan如下

```
|--Nested Loops(Left Semi Join, WHERE:([T2].[b]=[T1].[b] AND [T1].[a]=[T2].[a])) 
       |--Table Scan(OBJECT:([T1])) 
       |--Table Scan(OBJECT:([T2]))
```
从SQL Plan中可以看出，子查询和父查询之间有相关性，因为子查询中有谓词T2.b = T1.b。

> 这个去相关看得迷迷糊糊的，不太懂，还需要再看看。

# 本周后面的计划(针对OceanBase2.x)
- 学习Oceanbase的并行执行细节。
- 学习Oceanbase的SQL调优细节。
- 学习Oceanbase性能监控方面的细节。
- 和晓楚学长讨论，对实习项目有初步的需求规划，并加以完善，写出需求文档。
- 对实习项目的系统功能、接口初步设计，写出功能文档。







#### v$sql_audit 记录每一个sql请求的执行情况
按照什么排序？如何查到指定的sql语句情况

开启alter system set enable_sql_audit = true;才可以

如果发现sql_audit会淘汰你的sql，需要调大一点sql_audit的内存

alter system set sql_audit_memory_limit = '4G';

记录每条sql的执行情况，plan_id,参数化后的sql_id

只能记录整条sql的执行状况，很笼统，不能看出是否是分布式的执行计划。通过plan_id去plan_cache_plan_stat中查找具体的plan。

1. SVR_IP ip地址，分布式并行执行的话是什么？
2. SVR_PORT 端口号
3. REQUEST_ID 请求ID，干什么用的？
4. SQL_EXEC_ID ？？？
5. TRACE_ID trace的id，可以通过trace获得
6. SID  ？？？
7. CLIENT_IP 执行SQL的客户端IP
8. CLIENT_PORT 客户端端口
9. TENANT_ID  租户ID
10. TENANT_NAME  租户名
11. USER_ID 用户ID
12. USER_NAME 用户名
13. USER_CLIENT_IP 用户客户端IP
14. DB_ID  数据库ID？
15. DB_NAME 数据库名？
16. SQL_ID  SQL参数化后的ID
17. QUERY_ID    查询ID，什么用？
18. PLAN_ID 计划ID
19. AFFECTED_ROWS 影响行数
20. RETURN_ROWS 返回行数
21. PARTITION_CNT 分区数
22. RET_CODE 返回码
23. QC_ID  Query Coordinate ID，并行执行之后，协调算子的ID，协调算子协调job应该发往哪些节点。
24. DFO_ID 其实就是执行每个job的job_id
25. SQC_ID 其实就是taskID，多线程执行的。
26. WORKER_ID ？？
27. EVENT 最长等待时间名
28. P1TEXT 等待事件参数1
29. P1 等待事件1
30. P2TEXT 等待事件参数2
31. P2 等待事件2
32. P3TEXT  等待事件参数3
33. P3  等待事件3
34. LEVEL 等待事件的LEVEL级别
35. WAIT_CLASS_ID   等待事件的所属class id， classId是什么？
36. WAIT_CLASS# 等待事件的所属class下标
37. WAIT_CLASS  等待事件的所属class名称
38. STATE   等待事件的状态
39. WAITE_TIME_MICRO    该等待事件所等待的时间（us）
40. TOTAL_WAIT_TIME_MICRO 过程中总共的等待时间
41. TOTAL_WAITS 执行过程中总的等待次数
42. RPC_COUNT   发送RPC次数
43. PLAN_TYPE   执行计划类型 local、remote、distribute
44. IS_INNER_SQL    是否是内部请求sql
45. IS_EXECUTOR_RPC 当前请求是否是rpc
46. IS_HIT_PLAN 是否命中plan cache
47. REQUEST_TIME 开始执行时间点
48. ELAPSED_TIME    接收到请求到请求结束，消耗的总时间
49. NET_TIME 发送rpc到接收到请求的时间
50. NET_WAIT_TIME 接收到请求到进入队列时间
51. QUEUE_TIME 请求在队列中等待的时间
52. DECODE_TIME 出队列后decode所用时间
53. GET_PLAN_TIME 开始执行到获得到sql plan的时间
54. EXECUTE_TIME plan执行消耗的时间
55. APPLICATION_WAIT_TIME 所有application类事件的总时间
56. CONCURRENCY_WAIT_TIME 所有并发类事件的总时间
57. USER_IO_WAIT_TIME 所有USER_IO类事件的总时间
58. SCHEDULE_TIME 所有schedule类事件的总时间。
59. ROW_CHCHE_HIT 行缓存命中次数
60. BLOOM_FILTER_CACHE_HIT bloom filter缓存命中次数
61. BLOCK_CACHE_HIT 块缓存命中次数
62. BLOCK_INDEX_CHACHE_HIT 快缓存索引命中次数
63. DISK_READS  物理读取次数
64. RETRY_CNT 重试次数
65. TABLE_SCAN 是否为全表扫描
66. CONSISTENCY_LEVEL 一致性级别，各个级别有什么区别？
67. MEMSTORE_READ_ROW_COUNT memstore中读取的行数
68. SSSTORE_READ_ROW_COUNT ssstable中读取的行数
69. REQUEST_MEMORY_USED 该请求消耗的内存大小
70. EXPECTED_WORKWE_COUNT ？？？
71. USED_WORKER_COUNT ？？？
72. SCHED_INFO ？？？
73. FUSE_ROW_CACHE_HIT ？？？


#### v$plan_cache_stat 每个计划缓存的状态

计划缓存是根据租户隔离的，每个租户在每个observer上有一个计划缓存，每一个计划缓存是v$plan_cache_stat的一条记录。

1. tenant_id 租户ID
2. svr_ip 服务器ip，这个缓存表在哪个服务器
3. svr_port 服务器端口
4. sql_num 缓存的sql条数
5. mem_used 已经占用内存大小
6. mem_hold ？？？
7. access_count 进该plan_cache的次数
8. hit_count 命中该plan_cache的次数
9. hit_rate 命中率
10. plan_num ,plan的总数
11. mem_limit ,该plan_cache的内存上限
12. hash_bucket ,plan_cache的hash map的bucket个数
13. stmtkey_num stmtkey的个数。。和sql_num的关系是什么？？

#### v$plan_cache_plan_stat 

记录中包括这个计划是哪个租户，哪个ip地址，哪个port的计划缓存。

这个视图保存了每个plan对应的参数化后的sql，并且记录了生成这个plan的第一个sql的完整带参数语句。

这里的sql_id是生成这个plan的sql经过加密后的结果。

主键是(plan_id,tenant_id),可以通过这个plan_id和tenant_id, 查找v$plan_cache_plan_explain中对应的物理执行计划。

1. tenant_id 租户ID
2. svr_ip 服务器的ip
3. svr_port 服务器端口
4. plan_id ，plan的id
5. sql_id ，sql的id
6. type，plan的类型
7. is_bind_sensitive 该计划是否需要打开acs？？？？
8. is_bind_aware 该计划已经打开了acs？？
9. db_id 数据库的id
10. statement 参数化后的sql语句
11. query_sql 实际的查询sql
12. special_params 特殊的参数？？？
13. param_infos 参数信息？？
14. sys_vars 影响计划的系统变量的值
15. plan_hash ，plan的hash值
16. first_load_time， 该plan第一次被加载的时间
17. schema_version， schema的版本号
18. merged_version， 当前缓存的plan对应的合并版本号？？
19. last_active_time， 最后一次执行时间
20. avg_exe_usec，平均执行时间
21. slowest_exe_time 最慢执行时间
22. slowest_exe_usec，最慢执行时间戳？？
23. slow_count，当前plan成为慢查询次数？？
24. hit_count，被命中次数
25. plan_size，plan的大小
26. executions， 执行次数？？？和命中次数真么关系？
27. disk_reads， 所有执行物理读次数
28. direct_writes，所有执行写盘的次数
29. buffer_gets， 所有执行逻辑读的次数
30. application_wait_time 所有执行所有application类事件的总时间
31. concurrency_wait_time 所有执行所有concurrency类事件的总时间
32. user_io_wait_time 所有执行所有user_io类事件的总时间
33. rows_processed 所有执行选择的结果行数或执行更改表中的行数
34. elapsed_time 所有执行接收到请求到执行结束消耗时间
35. cpu_time 所有执行消耗的cpu时间
36. large_querys 被判断为大查询的次数
37. delayed_large_querys 被判断为大查询且被丢入大查询队列的次数
38. delayed_px_querys ？？？
39. outline_version ，outline版本号
40. outline_id ，outline的id，为-1表示不是通过绑定outline生成的计划
41. outline_data，计划对应的outline信息
42. asc_sel_info，当前acs计划对应的选择率空间？？？
43. table_scan ，表示该查询是否为主键扫描
44. evolution ，表示该执行计划是否在演进中
45. evo_executions，演进次数
46. evo_cpu_time，演进过程中总的执行CPU时间
47. timeout_count，超时次数
48. ps_stmt_id，prepare statement id？？？
49. sessid ？？？
50. temp_tables ？？？
51. is_use_jit 是否使用jit
52. object_type ？？？

#### v$plan_cache_plan_explain

可以通过tenant_id和plan_id查看某个plan的具体物理执行计划。
包含了 IP地址和PORT

1. TENANT_ID 租户ID
2. IP ，Plan所在IP地址
3. PORT，Plan所在端口号
4. PLAN_ID，plan的id
5. OPERATOR，operator名称
6. NAME，表的名称
7. ROWS，预估结果行
8. COST，预估代价
9. PROPERTY，对应operator的信息 

#### sql trace 提供上一次sql请求的各过程执行信息。

set ob_enable_trace_log = 1;

只能查看最近一条执行sql的具体执行情况，我在实现的时候可能要采样最近的执行sql，设计一个缓存。


### Explain 相关

explain不真正执行sql，只是统计信息的判断，这种有什么意义？

需要哪些视图？sql_audit、plan_cache_plan_stat、trace？

需要哪些功能？















## 第一章 React 开发环境准备

### 1.1 基本步骤

1. [Node 官网](https://nodejs.org/en/download/) 下载node，其中会顺带下载好npm
2. [React 官方文档](https://reactjs.org/docs/getting-started.html) 查阅资料
3. `npm install -g create-react-app` 下载React脚手架工具
4. `create-react-app todolist` 利用脚手架工具，新建一个React项目
5. `cd todolist`
6. `npm start` 开启React初始项目

### 1.2 工具推荐

1. React Chrome 插件[React Developer Tools](https://chrome.google.com/webstore/detail/react-developer-tools/fmkadmapgofadopljbjfkapdkoienihi?utm_source=chrome-ntp-icon)（开发环境下查看组件变化）
2. Redux Chrome 插件[Redux DevTools](https://chrome.google.com/webstore/detail/redux-devtools/lmhkpmbekcpmknklioeibfkpmmfibljd?utm_source=chrome-ntp-icon)
3. 代理服务器[Charles](https://www.charlesproxy.com/)（本地 mock 数据）

### 1.3 常用模块

1. 安装 [react-transition-group](https://github.com/reactjs/react-transition-group) `npm install react-transition-group --save`（动画）
2. 安装 [axios](https://github.com/axios/axios)  `npm install axios`（Ajax 请求）
3. 安装 [Ant Design of React](https://ant.design/index-cn) `npm install antd --save`
4. 安装 [Redux](https://redux.js.org/) `npm install --save redux`
5. 安装 [Mock.js](https://github.com/nuysoft/Mock) `npm install mockjs` （Mock 数据测试）
5. 安装 [Redux Thunk](https://github.com/reduxjs/redux-thunk) `npm install redux-thunk`
6. 安装 [Redux Saga](https://github.com/redux-saga/redux-saga) `npm install --save redux-saga`
7. 安装 [React Redux](https://github.com/reduxjs/react-redux) `npm install --save react-redux`
8. 安装 [styled-components](https://github.com/styled-components/styled-components) `npm install styled-components`（使 CSS 模块化）
9. 安装 [immutable](https://github.com/immutable-js/immutable-js) `npm install immutable`（封装 state ，确保不改变）
### 1.4 个别工具使用说明

#### 1.4.1 Mock.js 基础使用

```js
// mock/index.js
import Mock from 'mockjs';

import getListData from './get/api_get_list';
import getTestData from './get/api_get_test';

export default Mock.mock('/api/get/list', 'get', getListData)
.mock('/api/get/test', 'get', getTestData);

// mock/get/api_get_test
export default {
    name: 'Kiyonami',
    age: 20
}

//最后在 src 下的 index.js 引入即可！（一处引用就可以！）
import './mock';
```

#### 1.4.2 Redux Thunk 配置

参照 [Exercise-Project/todos/src/store/index.js](https://github.com/514723273/Exercise-Project/blob/master/todos/src/store/index.js) 和 [Redux DevTools Extension](https://github.com/zalmoxisus/redux-devtools-extension)

#### 1.4.3 iconfont 网站使用

1. 登录账号，新建项目
2. 搜索想要的图标，点击购物车，添加至项目
3. 下载到本地
4. 解压，选择五个文件放入项目
  - iconfont.css
  - iconfont.eot
  - iconfont.svg
  - iconfont.ttf
  - iconfont.woff

#### 1.4.4 ReduxDevTools 插件在代码中的配置

```js
import { createStore, /*applyMiddleware,*/ compose } from 'redux';
import reducer from './reducer';

// import createSagaMiddleware from 'redux-saga';
// import sagas from './sagas';

// const sagaMiddleware = createSagaMiddleware();

const composeEnhancers =
  typeof window === 'object' &&
  window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ ?   
    window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__({}) : compose;

const enhancer = composeEnhancers(
    // applyMiddleware(sagaMiddleware),
);

//第二个参数是为了配置浏览器 redux 扩展
const store = createStore(reducer, enhancer);
//Before running a Saga, you must mount the Saga middleware on the Store using applyMiddleware
//sagaMiddleware.run(sagas);

export default store;
```

### 1.5 问题

#### 1.5.1 npm start 启动报错

[中途出现的问题及解决方案](https://github.com/facebook/create-react-app/issues/6985)
```
create-react-app my-app
cd my-app
npm install react-scripts@2.1.8     //降级为 react scripts@2.1.8 解决了这个问题，而不需要处理环境变量
npm start
```

#### 1.5.2 create-react-app 慢

虽然平常使用cnpm来代替npm,但也只是使用新的指令而已，所以需要整体换源。

`npm config set registry https://registry.npm.taobao.org`

## 第二章 React 生命周期函数

生命周期函数指在某一时刻组件**会自动调用执行**的函数

![](https://raw.githubusercontent.com/514723273/.md-Pictures/master/20190604203050.png)

### 2.1 完整生命周期

#### 2.1.1 Initialization 初始化

设置props和state，这是在constructor中完成的。

#### 2.1.2 Mounting 挂载

- componentWillMount 在组件**即将**被挂载到页面的时刻自动执行
- render 页面挂载执行
- componentDidMount 挂载结束后执行

#### 2.1.3 Updation 组件更新

1. props 独有（多一个）：
- componentWillReceiveProps 当一个组件从*父组件**接受参数*，如果这个组件第一次存在于父组件中，不会执行；如果这个组件之前已经存在于父组件中，才会执行。

2. props和state更新共有的函数：（就是 state 全部）
- shouldComponentUpdate 组件被更新前，自动执行，返回的是一个bool值。你这个组件需要更新吗？？？返回true需要，**返回false不更新，下面函数都不会执行**。
- componentWillUpdate 组件被更新前，它会被自动执行，但是他是在 shouldComponentUpdate 之后执行。
- render 重新渲染DOM。
- componentDidUpdate 件更新结束后执行。

#### 2.1.4 Unmount 卸载

- componentWillUnmount 当这个组件即将被从页面移除的时候，自动执行。（例如 todolist 删去其中一个 todo 项）

### 2.2 React生命周期函数的使用场景

#### 2.2.1 阻止无意义的渲染

在TodoList中，每次输入input框，父组件render，下面的TodoItem也会跟着被render一次。（参考 Diff 算法同级比较思想，找到一个节点不同，底下所有的子节点都替换）但是**TodoItem没有做任何改变，却要被无意义的渲染，这是浪费性能**。

所以使用`shouldComponentUpdate(nextProps, nextState) {}`函数来阻止这无意义的渲染。接受两个参数 nextProps 和 nextState 表示我接下来 props 和 state 会变成什么样。
```js
//通过控制判断内容是否被更改，来判断是否需要重新渲染。
shouldComponentUpdate(nextProps, nextState) {
	if(nextProps.content !== this.props.content) {
		return true;
	} else {
		return false;
	}
}
```

#### 2.2.2 Ajax 数据请求位置

Ajax请求该放在哪？

如果放在 render 函数内最前面，总是会在输入的时候，不停地调用 render ，无意义地向服务器请求数据，其实只要请求一次就好了。

在生命周期中，执行一次的可以放在 componentDidMount() （最合适！），挂载结束后获取。

```js
//用axios来发送ajax请求。
componentDidMount() {
	axios.get('/api/todolist')
	.then(res => {
    //在拿到返回数据结果后，再 setState 初始化列表，又会重新 render 一遍列表
    this.setState(() => ({
      list: [...res.data]
    }));
  })
	.catch(()=>{alert('error')})
}
```

## 第三章 虚拟 DOM

### 3.1 三种方式的演变与比较

#### 3.1.1 方式一

1. State 数据
2. JSX 模板
3. 数据 + 模板 结合，生成真实的DOM（页面显示的DOM），来显示
4. state 发生改变
5. 数据 + 模板 结合，生成真实的DOM，替换原始的DOM

缺陷：

第一次生成了一个完整的DOM片段

第二次生成了一个完整的DOM片段

第二次的DOM替换第一次的DOM，非常耗性能

#### 3.1.2 方式二

1. State 数据
2. JSX 模板
3. 数据 + 模板 结合，生成真实的DOM，来显示
4. State 发生改变
5. *数据 + 模板 结合，生成真实的DOM，（并不直接替换原始的DOM，存储在内存）*
6. *新的DOM（DocumentFragment）和原始的DOM做比对，找差异*
7. *找出input框发生的变化*
8. *只用新的DOM中的input元素，替换掉老的DOM中的input元素*

缺陷：

节省了DOM替换的性能，但是损耗了DOM对比的性能，虽然性能有所提升，但是不大。

#### 3.1.3 方式三

1. State 数据
2. JSX 模板
3. 数据 + 模板 结合，生成虚拟DOM（**虚拟DOM就是一个JS对象，用它来描述真实的DOM**），如下
```js
[标签，属性对象，内容]
['div', {id: 'abc'}, ['span', {}, 'hello world']]
```
4. 用虚拟DOM的结构生成真实的DOM，来显示
```js
<div id = 'abc'>
  <span>
    Hello world!
  </span>
</div>
```
5. state 发生变化
6. 数据 + 模板 生成新的虚拟DOM（极大的提升了性能）（此时有两份虚拟DOM）
['div', {id: 'abc'}, ['span', {}, 'bye bye']]
7. 比较原始虚拟DOM和新的虚拟DOM的区别，找到区别是span的内容（极大的提升了性能）
8. 直接操作DOM，改变span中的内容

优点：

节省了真实DOM的创建，节省了真实DOM的对比，取而代之，是创建了JS对象，对比的也是JS对象。
（**JS比较JS对象不怎么消耗性能，但是操作真实的DOM十分消耗性能**）

### 3.2 深入了解虚拟 DOM

例如
```js
return <div>item</div>
```

中间的<div></div>只是一个JSX模板，并不是真实的DOM，也不是虚拟DOM。

JSX -> JS 对象（虚拟 DOM） -> 真实的 DOM

`return React.createElement('div', {}, item);` 这是一个更加底层的实现，效果一样。

也就是说即使没有JSX语法，也能实现虚拟DOM，只是比较麻烦。

再例如
```js
return <div><span>item</span></div>
//等同于
return React.createElement('div', {}, React.createElement('span', {}, item))
```
再深入可得，JSX -> *createElement* -> JS 对象（虚拟 DOM）-> 真实的 DOM

### 3.3 虚拟 DOM 的优点

1. 性能提升了， DOM 的比对变为 JS 对象的比对。
2. 它使得跨端应用得以实现。React Native。

（在浏览器中渲染真实的DOM没有问题，但是IOS、Android是不存在DOM这个概念的，无法被使用！但是转化为虚拟DOM，一个JS对象，可以在各平台被识别，在浏览器渲染真实的DOM，在移动平台渲染各种原生组件。）

### 3.4 虚拟 DOM 中的 Diff 算法

如何比较两个虚拟DOM的内容就涉及到Diff算法。

#### 3.4.1 Diff 的时机

![](https://raw.githubusercontent.com/514723273/.md-Pictures/master/20190605143116.png)

虚拟DOM的比对是在数据发生了变化之后。state改变，（props的改变实则父组件的state的改变），所以都是先调用了`setState`方法。

setState 是异步的，为了提升性能。比如连续调用三次 setState ，比对三次，更新三次，这样比较浪费性能，所以在极短的时间，**被合并为一次 setState**，一次比对虚拟DOM一次更新真实的DOM，省去额外两次。

#### 3.4.2 Diff 同级比较思想

![](https://raw.githubusercontent.com/514723273/.md-Pictures/master/20190604155508.png)

同级比对，如果在节点比较就发现不同的话，下面就不会继续比较了，直接全部重新生成替换所有子节点。（虽然这样看上去很浪费性能，但是算法简单，提升效率。）

#### 3.4.3 Key 值的重要性

![](https://raw.githubusercontent.com/514723273/.md-Pictures/master/20190604160604.png)

例如一个列表（如图）：
- 第一种情况：没有 Key 值，当数据发生改变，生成新的虚拟 DOM （第二排的小圆球），使用 Diff 算法与原虚拟 DOM 对比的时候，因为没有名字，不能很快地将其一一对应。（类似使用两层循环的比较，使其一一对应，十分消耗性能）。
- 第二种情况：当每项都拥有 Key 时，拥有自己的名字，能很快地建立联系，一一对比，找出不同。提高了虚拟DOM的比较性能。

*再提个问题，之前为什么不能用数组 Index 做 Key 值？*

不能保证原始的虚拟DOM和新的虚拟DOM 的 key 值一致了。

比如原始数组元素 [a, b, c] ， Index 分别是 0, 1, 2 。

当删去了 a ， [b, c] 的 Index 就变成了 0, 1 。因为数据变化（ a 删去），所以生成新的虚拟 DOM ，此时生成的 DOM 的 Key 值是按 0, 1 生成的，与原来的虚拟 DOM 中的 1, 2 不一致了，无法直接建立联系了，失去了 Key 的意义。

所以要使用稳定的值作为 Key 值，例如 唯一 id 。

## 第四章 Redux 入门

### 4.1 Redux 概念简述

![](https://raw.githubusercontent.com/514723273/.md-Pictures/master/20190609202153.png)

Redux就是把组件数据放进一个公共区域进行存储。

组件改变数据就不需要传递了，改变store里面的数据，其他组件就会感知到数据的改变，再去取就能取到新数据。所以无论层次多深，流程都是统一的。

### 4.2 store 自己的理解

![store](https://raw.githubusercontent.com/514723273/.md-Pictures/master/store.png)

按步骤分析：

#### 4.2.1 新建 reducer 函数

参数为 (state, action)

#### 4.2.2 store = createStore(reducer)

 **将 reducer 置于内部，并且会马上执行一次 reducer**，此时会在内部存储一份 state 。
 
 此时执行 reducer 传入的参数分别为`state = defaultState, action = {type: "@@INIT"}`（此时是没有传入 action ，这个 action 为内部传入；defaultState 为默认值）

 （所以在组件的 constructor 中，第一次 this.state = store.getState() ，是将组件内的 state 指针指向 store 内部的 state）

#### 4.2.3 新建 action 对象

需要有 type 属性

#### 4.2.4 store.dispatch(action)

日常执行 reducer 函数，传递的参数为 store 内部的 state 和 此时作为参数的 action 。更新内部的 state 。

#### 4.2.5 store.subscribe(func)

注册 store 监听，当内部的 state 发生改变时，调用 func 。

#### 4.2.5 this.setState(store.getState())

一般是注册 store 监听时，组件传入的函数。`store.subscribe(() => {this.setState(store.getState())});`

将组件的 state 指针重新指向 store 内的新的 state 。（因为每次 store 内 state 变化是新拷贝一份旧的 state ，在拷贝后的做修改并返回）

### 4.3 UI 组件和容器组件

可以拆分为 UI 组件和容器组件：
- *UI 组件*负责组件渲染，内部一般只有一个 render 函数，使用无状态组件（函数组件）（性能高）来表示。
- *容器组件*负责数据交互，引用对应的容器组件将其渲染，将 UI 组件需要的内容通过属性传递（ UI 组件通过 props 接收）（引入 store ，通过 dispatch 传递参数，在 reducer 中逻辑处理。）

## 第五章 Redux Thunk and Redux Saga

### 5.1 Redux Thunk 中间件

如果遇到异步请求或者非常复杂的逻辑，我们希望把它移到其他地方进行统一处理，否则会使一个组件看起来过于臃肿。

action 中返回的可以不是一个对象了，可以是一个函数。

具体使用可以参考 [Exercise-Project/todos](Exercise-Project/todos/src/store/actionCreators.js)

### 5.2 什么是中间件

![Redux Data Flow](https://raw.githubusercontent.com/514723273/.md-Pictures/master/20190610074946.png)

中间是指 action 和 store 的中间。中间件是对 dispatch 做了一次升级，可以对传入的参数进行判断，例如是对象直接传入 store ，是函数先执行，再传入。
```js
// actionCreators.js
/**
 * redux thunk
 */
//使用 redux-thunk 后， action 可以是函数
export const getInitList = () => {
    //返回函数 （虽然可以再简写 不用 return 用小括号
    //虽然没有 store ，但是这个函数会接收到 dispatch 方法
    return (dispatch) => {
        axios.get('/api/get/list').then((res) => {
            const action = getInitListAction(res.data);   //返回的函数继续执行，还是会创建 action 传入 store
            dispatch(action);
        });
    }
}
```

### 5.3 Redux Saga 中间件

Redux Saga 和 Redux Thunk 用途一致，但流程不同：
- redux saga 和 redux thunk 的不同是，thunk 是通过传入的 action 是函数还是对象自动分配；
- 而 saga 不仅可以将 action 传入 reducer 也可以传入 saga。，我们先创建一个新的 action 对象，传入到 saga 中，再执行 InitList 的 action 。

```js
// saga.js
import { takeEvery, put } from 'redux-saga/effects';
import { GET_INIT_LIST } from './actionTypes';
import { getInitListAction } from './actionCreators';
import axios from 'axios';

function* getInitList() {
    try {
        const res = yield axios.get('/api/get/list');
        const action = getInitListAction(res.data); //此时又使用 初始化list的action
        yield put(action);
    } catch(e) {
        console.log('/api/get/list 网络请求失败');
    }

}

// 必须是 generator 函数
// 不仅可以在 reducer 接收到 action ，saga 也可以
function* saga() {
    yield takeEvery(GET_INIT_LIST, getInitList);
}

export default saga;
```
