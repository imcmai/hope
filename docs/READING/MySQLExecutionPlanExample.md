## is null、is not null 、!=能否使用索引
如今一直流传一种说法，这几种方式都不能使用到索引，我们来自己测试一下
### 表DDL信息
```code
CREATE TABLE `k1` (
  `id` int(11) NOT NULL,
  `key1` varchar(255) DEFAULT NULL,
  `key2` int(11) DEFAULT NULL,
  `key3` decimal(10,0) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `key2` (`key2`) USING BTREE,
  KEY `key1` (`key1`) USING BTREE,
  KEY `key3` (`key3`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
### 执行sql
```code
EXPLAIN select * from k1 WHERE key1 is null;
EXPLAIN select * from k1 where key1 is not null;
EXPLAIN select * from k1 where key1 !='a';
```
### 执行计划
```code
"1"	"SIMPLE"	"k1"		"ref"	"key1"	"key1"	"768"	  "const"	"1"	"100"	"Using index condition"
"1"	"SIMPLE"	"k1"		"range"	"key1"	"key1"	"768"		      "1"	"100"	"Using index condition"
"1"	"SIMPLE"	"k1"		"range"	"key1"	"key1"	"768"		       "2"	"100"	"Using index condition"
```
### 结论
几种方式都可以使用到索引，用不用索引的依据到底是什么呢,依据有两点，一是查找二级索引的成本，二是回表的成本，
举个极端例子，比如二级索引全部符合要求，这种情况不如直接扫描聚簇索引，还减少了回表的操作，所以mysql优化器
在执行sql前，会根据你的查询方式使用index dive的方式去得到一个二级索引匹配范围的值,或者用其他方式得到一个模糊值,
然后根据这个值计算成本,大于某个比例时才不会使用索引,所以mysql到底用不用索引，取决于成本！