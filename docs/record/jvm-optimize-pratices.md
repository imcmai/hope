收到某个服务的告警，服务堆内存占用超80%, 看下告警的日志，持续的时间很长，需要定位问题。
shell看下堆内存的分布
```code
root@mwvpn:/# jmap -heap 1
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
初步分析问题是由老年代过大并且没有回收引起的，堆的总内存3800m，老年代使用的3100M已经占用了80%以上了。  
我们看下jvm相关的启动参数
```code
root@mwvpn:/# jinfo -flags 1
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.101-b13
Non-default VM flags: -XX:CICompilerCount=4 -XX:InitialHeapSize=790626304 -XX:MaxHeapSize=3984588800 -XX:MaxNewSize=536870912 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=536870912 -XX:OldSize=253755392 -XX:+P
rintGC -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC
Command line:  -Xmn512m -Xmx3800m -Xloggc:/data/logs/gc.log -Dserver.port=80
```
关键地方 **-XX:+UseParallelGC** 这里标注了GC的垃圾回收器，ParallelGC回收器对于老年代的回收采用的是Parallel Old, 可以通过gc的log文件看到老年代的收集器， 也可以通过命令来验证
```code
root@mwvpn:/# java -XX:+PrintFlagsFinal -version | grep "UseParallelOldGC"
     bool UseParallelOldGC                          = true                                {product}
java version "1.8.0_101"
Java(TM) SE Runtime Environment (build 1.8.0_101-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.101-b13, mixed mode)
```
或
```code
root@r-mwvpn:/# jinfo -flag UseParallelOldGC 1
-XX:+UseParallelOldGC
```
在新生代的对象需要移动到老年代，并且老年代放不下时，才会触发Major GC ,当没有触发回收的时候，老年代的内存占用居高不下，所以就频繁的在告警。  
但是老年代之所以占用的内存这么高，很可能是因为survivor区过小导致，survivor区是eden在回收后，对象如果依旧存在引用，会把对象转入survivor区，survivor如果放不下才会转入到old区
###  为什么ps_survivor_space这么小?   
通过之前的jmap命令我们可以看到
```code
SurvivorRatio = 8

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
```
Eden和From、To的比例应该是 8:1:1 ，但是这里，From和To都是6M，出现这么大的偏差，很明显这个配置没有生效，我们参考RednaxelaFX在社区的回答
> HotSpot VM里，ParallelScavenge系的GC（UseParallelGC / UseParallelOldGC）默认行为是SurvivorRatio如果不显式设置就没啥用。显式设置到跟默认值一样的值则会有效果  

为什么这样设计?
> 因为ParallelScavenge系的GC最初设计就是默认打开AdaptiveSizePolicy的，它会自动、自适应的调整各种参数

这里也就明白了survivor区为什么这么小了，是由AdaptiveSizePolicy来调控的
为什么这个自适应策略会将survivor区调整的这么小？
我们先找到oracle的文档[The Parallel Collector](https://docs.oracle.com/en/java/javase/11/gctuning/parallel-collector1.html#GUID-DCDD6E46-0406-41D1-AB49-FB96A50EB9CE)，文档里表述了AdaptiveSizePolicy会根据MaxGCPauseMillis和GCTimeRatio两个值来决定堆的大小，很可能就是应用程序的gc停顿时间达不到要求，导致缩小了新生代，但是我们又强制指定了新生代为500m，所以survivor区就变得很小了。
两个参数的
```code
-XX:MaxGCPauseMillis=18446744073709551615 
-XX:GCTimeRatio=99
```
很多文章MaxGCPauseMillis说的默认200ms指的都是g1的回收器哈，这里ParallelScavenge的默认值是无符号Long类型的最大值，这个值没什么参考意义，AdaptiveSizePolicy还是按照GCTimeRatio的比例来调整的，也就是gc停顿时间占1%
### 解决方案
显示指定-XX:SurvivorRatio=8
#### 补充
在实践中即使将SurvivorRatio显示设置到跟默认值(SurvivorRatio = 8)一样,也不会生效，需要在显示指定survivor区的比例时，同时关闭自适应策略
```code
-XX:SurvivorRatio=8 -XX:-UseAdaptiveSizePolicy
```
