#基于Oracle的SQL优化读书笔记
----------------------------

##Oracle的优化器
SQL语句的执行过程大体可以描述为：  
SQL语句 → 解析 → 查询转换 → 查询优化器(RBO|CBO)→执行计划→实际执行→ 返回结果

###1.RBO(Rule-Based Optimizer)
  RBO的规则是硬编码在Oracle代码中的。RBO将各类型的执行路径划分为15个等级，从1到15.1执行效率最高，15最低。 1对应single row by rowid，15对应 full table scan。从10g开始RBO不载支持，但可以通过修改优化器模式继续使用。  
  只要出现了一下情况，即便使用RULE Hint，Oracle依旧不会使用RBO。  

- 目标SQL中涉及的对象有IOT(Index Organized Table)
- 目标SQL中设计的对象有分区表
- 使用了并行查询或者并行DML
- 使用了星型连接。
- 使用了Hash连接。
- 使用了函数索引  
 
如果出现了两条或者两条以上等级值相同的执行路径，RBO会一句SQL涉及对象在数据字典缓存中的缓存顺序和个对象在SQL中出现的先后顺序综合判断。  

**RBO的缺点：** 没有考虑对象的实际情况，比如数据量大小，数据分布情况。比如: 
 
	select * from a where a.foo = 'bar';
当foo上包含索引而且有1kw的数据foo字段均为 bar时，RBO会走索引。但此时使用索引块扫描后回表，效率并不如全表扫描来得好。  
###2.CBO(Cost-Based Optimizer)
CBO的成本是基于目标SQL设计的表、索引、列等对象的统计信息计算出来的。根据统计信息估算出来的目标SQL对应执行步骤的I/O、CPU、网络资源的估算值。

**CBO的几个概念**

- Cardinality(number of elements of the set) 书中解释为集的势。结果集包含的记录数，标识目标SQL具体执行步骤的执行结果半酣记录数的估算。
- Selectivity 选择率 指世家指定条件后返回结果集的记录数占未施加记录数的比率。
- Transitivity 可传递性 对原目标SQL做简单的等价改写,在原目标SQL中加上根据该SQL现有的谓词条件推算出新条件，提供更多的执行路径给CBO选择。
    1. 简单谓词传递: t1.c1 = t2.c1 and t1.c1 =10 传递后为 t1.c1 = t2.c1 and t1.c1 =10 and t2.c1=10
    2. 连接谓词传递: t1.c1 = t2.c1 and t2.c1 = t3.c1 传递后 t1.c1 = t2.c1 and t2.c1 = t3.c1 and t1.c1 = t3.c1
    3. 外连接谓词传递: t1.c1 = t2.c1(+) and t1.c1 =10 传递后 t1.c1 = t2.c1(+) and t1.c1 = 10 and t2.c1(+) =10
    
 **CBO的局限性：**  
  
- CBO会默认目标SQL语句where条件中出现的各列之间是独立的，没有关联关系
- CBO会假设所有的目标SQL都是单独执行的，并且互不干扰。比如有些SQL将索引读取缓存了。
- CBO对直方图统计信息有诸多限制 比如文本字段只截取头32字节
- CBO在解析多表关联的目标SQL时，可能会露选正确的执行计划。如多个表连表查询时，执行路径总数会呈几何奇数增长，所以不可能遍历所有可能的执行路径。

##优化器的基础知识
###1.优化器的模式
- RULE       =  RBO
- CHOOSE  9i的默认值，如果对象有统计信息则CBO，如果没有RBO
- FIRST_ROWS__*n*
- FIRST_ROWS
- ALL_ROWS   10g的默认值
###2.结果集
执行计划中Rows标识的估算值
###3.访问数据的方法
访问数据一般分为两种： 一种直接访问表，另一种是先访问索引，然后回表(如果访问索引可以获取数据则不用回表)。
####访问表的方法：
1. 全表扫描。从第一个区的第一个块开始扫描，直至高水位线。全表扫描会使用多块读。但根据表的数据量大小变化，执行效率会有很大不确定性。对表中的数据做delete操作时，不会影响高水位线。
2. ROWID扫描。 ROWID是Oracle中的数据行揭露所在的屋里存储地址，ROWID实际是Oracle中数据块里的记录一一对应。
####访问索引的方法：
1. 索引唯一性扫描。针对唯一性索引的扫描，where条件是等值的目标SQL，至多返回一条记录。
2. 索引范围扫描
3. 索引全扫描。
4. 索引快速扫描
5. 索引跳跃式扫描
###4 表连接
表连接包含以下几点:   
1. 表连接顺序 ：不管有多少个表连接，Oracle都会先做两两连接，知道目标SQL全部连接。要决定先连接哪两个，要决定谁做驱动表(inner table)。  
2. 表连接的方法：排序合并连接、嵌套循环连接、哈希连接和笛卡尔连接。  
3. 访问单表的方法：在访问单表时，走索引还是全表。
4. 表连接的类型：  

- 内连接(Inner Join) 连接结果只包含完全满足连接条件的记录。只要where条件中没有标准SQL中定义或者Oracle中自定义的外连接的关键字， 标准SQL中的 left outer join,right outer join,full outer join,Oracle中自定义的(+) 就是内连接。  
 
		slect t1.coll,t1.col2,t2.col3 from t1,t where t1.col2 = t2.col2;

  这是Oracle自己的写法，标准SQL中用Join on 或者 join using  

	t1 join t2 on
	t1 join t2 using (多个条件用逗号分隔)  使用join using的时候不能写表名或表的别名
nature join 自动使用同名的column，不建议使用  
- 外连接(Outer Join)除了包含完全满足连接条件的记录之外还包含驱动表中锁不满足的连接条件的记录。 标准SQL中国的外连接：  

	left outer join
	right outer join
	full outer join 
对于left outer join :  
 
	a left outer join b on   
此时包含所有满足内连接的记录，a中不满足的记录也会在结果集中，结果集中b部分的内容为null。 右连接同理。  

    a full outer join b on  
近似于：  

	a left outer join b on   
	union 
	a right outer join b on

在Oracle中





 
  

- 
- 




