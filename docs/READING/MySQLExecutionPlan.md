# mysql执行计划
mysql在基于成本优化后会生成执行计划
## id
为每个select分配一个唯一的id，比如子查询可能就会分配两个或两个以上id，连接查询(不包括UNION)就是一个，但是在有些情况下子查询会被优化为连接查询去执行，id相同靠前的为驱动表(可以成为基表，或者主表),靠后的为被驱动表
## select_type
### simple
单表查询或者连接查询属于simple，不包括子查询和union
### primary
union,unionall,子查询，最左属于primary
### union
相对于primary,其他属于union，举例说明
```sql
EXPLAIN select key1 from k1 UNION all select name from test
"1"	"PRIMARY"	"k1"		"index"		"key1"	"768"		"1"	"100"	"Using index"
"2"	"UNION"	"test"		"ALL"					"1"	"100"	
```
- union result
生成临时表用于union的去重，查询这个临时表的select_type就是unionresult
## possible_keys
可能用到的索引列
## keys
实际索引列
## keys_len
索引列长度，其实是固定的，放在这里主要为了用于使用到联合索引时计算出使用到了几个索引列
## ref
当使用等值查询时，用于描述被等值的列或者常数
## rows
全表扫描时，rows是大致需要扫描的行数， 使用索引时代表大约匹配的索引记录数
## type
### system
基本用不到，就不讲了
### const
使用主键等值时，type会被标志为const类型，因为聚簇索引生成的B+tree可以直接通过主键定位到叶节点的实际记录，
唯一索引也是一样，只需定位一条记录然后回表(如果只查询索引列，则连回表都可以省略)到聚簇索引查询实际记录，如果是多列构成，则必须保证
全部都是等值比较，使用唯一索引时，如果比较的是 is null这种值则不会使用const，而是ref
### ref
普通二级索引的等值比较由于不是唯一的，所以查询的代价取决于匹配的范围有多少条记录，匹配记录越少回表的效率越高
所以mysql还是会尽量使用索引，这种类型就是ref，如果是多个索引列，也需满足最左的连续索引列都是等值
### ref_or_null
同时对一个索引列使用等值和is null(唯一索引也一样，因为索引并不限制null值)，会去查两次索引树然后回表到聚簇索引
### range
 对于区间的查询操作 如:column>=value
### index
直接遍历二级索引，无需回表操作，称之为index，也就是查询列包含在索引列中
### all
直接扫描聚簇索引，就是all类型
### index_merge
一般情况下，执行一个查询，只会用到一个二级索引，特殊情况下会用到多个二级索引，使用交集或并集等方式合并索引，这种情况就是index_merge
### 注
以上只是描述假如采用二级索引，会出现的几种类型，具体采用哪种方式，还要取决于索引回表的代价，代价如果很高，那么是会全表扫描的，