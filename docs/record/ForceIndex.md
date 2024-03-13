## 指定sql使用的索引
```code
select * from table force index(index)
```
mysql优化器会根据估算的一个成本选择使用或者舍弃掉使用某个索引，但是有时候这个决定并不是正确的，
可以采用 force index 强制使用某个索引
## 扩展
1. use index  推荐mysql使用某个索引， 忽视其他的索引
2. ignore index 忽略某些索引
3. force index 强制使用某个索引