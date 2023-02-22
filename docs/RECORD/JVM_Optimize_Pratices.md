## 一次堆内存高占用的问题排查
同事说线上收到某个服务普罗米修斯的告警，服务堆内存占用超80%, 看下告警的日志，持续的时间很长，让我帮忙协助处理一下， 首先查看目前的堆分布
```code
root@8120c-8789bc78b-mwvpn:/# jmap -heap 1
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.101-b13

using thread-local object allocation.
Parallel GC with 10 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 3984588800 (3800.0MB)
   NewSize                  = 536870912 (512.0MB)
   MaxNewSize               = 536870912 (512.0MB)
   OldSize                  = 253755392 (242.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 524288000 (500.0MB)
   used     = 441591296 (421.13427734375MB)
   free     = 82696704 (78.86572265625MB)
   84.22685546875% used
From Space:
   capacity = 6291456 (6.0MB)
   used     = 4620448 (4.406402587890625MB)
   free     = 1671008 (1.593597412109375MB)
   73.44004313151042% used
To Space:
   capacity = 6291456 (6.0MB)
   used     = 0 (0.0MB)
   free     = 6291456 (6.0MB)
   0.0% used
PS Old Generation
   capacity = 3363307520 (3207.5MB)
   used     = 3291755280 (3139.262466430664MB)
   free     = 71552240 (68.23753356933594MB)
   97.8725632558274% used

22892 interned Strings occupying 2421840 bytes.
```
可以定位到问题是由老年代过大并且没有回收引起的，堆的总内存3800m，老年代已经占用了80%以上了。  
我们看下jvm相关的启动参数
```code
root@8120c-8789bc78b-mwvpn:/# jinfo -flags 1
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.101-b13
Non-default VM flags: -XX:CICompilerCount=4 -XX:InitialHeapSize=790626304 -XX:MaxHeapSize=3984588800 -XX:MaxNewSize=536870912 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=536870912 -XX:OldSize=253755392 -XX:+P
rintGC -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC
Command line:  -Xmn512m -Xmx3800m -Xloggc:/data/logs/gc.log -Dserver.port=80
```
关键地方 **-XX:+UseParallelGC** 这里标注了GC的垃圾回收器，对于老年代的回收采用的是Parallel Old, 在新生代的对象需要移动到老年代，并且老年代放不下时，才会触发Major GC ,当没有触发回收的时候，老年代的内存占用居高不下，所以就频繁的在告警。  
我们能从上面发现几个问题呢？
### 1. -Xmn的含义
-Xmn指定了大小为512m，也就等于指定了新生代的大小为512M， 不会变大不会变小，固定就是这个值
### 2. 为什么ps_survivor_space这么小?  
我们知道,survivor区是eden在回收后，对象如果依旧存在引用，会把对象转入survivor区，但是这里的大小是6M，也就导致了大部分存活的对象如果放不下，就直接转入了Old区  
通过之前的命令我们可以看到
```code
SurvivorRatio            = 8
```
eden和survivor的比例是 8:1 ，那么为什么会出现这么大的偏差呢，很明显这个配置没有生效，我们参考之前RednaxelaFX的回答
> HotSpot VM里，ParallelScavenge系的GC（UseParallelGC / UseParallelOldGC）默认行为是SurvivorRatio如果不显式设置就没啥用。显式设置到跟默认值一样的值则会有效果  

为什么这样设计?
> 因为ParallelScavenge系的GC最初设计就是默认打开AdaptiveSizePolicy的，它会自动、自适应的调整各种参数

这里也就明白了survivor区为什么这么小了
### 解决方案
1. 调整-Xmn的大小
2. 显示指定-XX:SurvivorRatio=8
### 补充
在显示指定survivor区的比例时，需同时关闭自适应策略
```code
```-XX:SurvivorRatio=8 -XX:-UseAdaptiveSizePolicy
```
