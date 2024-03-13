## 查看堆内对象top50占用内存
```code
jmap –histo:live $pid | sort-n -r -k2 | head-n 50
```