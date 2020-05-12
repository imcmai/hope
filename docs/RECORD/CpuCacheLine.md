## 代码示例
```code
    public static void main(String[] args) {
        int[] arr = new int[64 * 1024 * 1024];
        //loop1
        for (int i = 0; i < arr.length; i++) arr[i] *= 3;
        //loop2
        for (int i = 0; i < arr.length; i+=16) arr[i] *= 3;
        //loop3
        for (int i = 0; i < arr.length; i+=1024) arr[i] *= 3;

    }
```
### 执行结果
1. 循环1和循环2执行时间几近相同
2. 循环3执行时间远低于1和2
## 造成结果的原因
cpu有主存和缓存之分，读取数据时，会把数据从主存放到缓存中处理，每一次读取一个cache line(64字节),上述的代码执行快慢的根本原因是从主存加载到缓存的次数不同，loop1每循环16次，需要加载一次cache line，loop2每次循环正好跨16个元素，也就是每次循环需要加载一次cache line, 而loop3因为循环的步长为1024，虽然每次也要访问cache line但是总共的加载cache line的次数远低于loop1和loop2。
## 代码示例补充
1. 步长1~16消耗时间基本一致，读取的都是同一块cache line,不会去主存中加载，真正消耗时间的地方在读取cache line到缓存中
2. 步长大于16之后，随着步长的增加，消耗的时间也在递减
> int类型4四字节，cache line64字节，可以存储16个int数据


