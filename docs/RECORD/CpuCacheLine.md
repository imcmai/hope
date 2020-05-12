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
cpu有主存和缓存之分，读取数据时，会把数据从主存放到缓存中处理，每一次读取一个cache line(64字节),上述的代码执行快慢的根本原因是从主存加载到缓存的次数不同，int类型占用4字节，loop1每循环16次(首次也需加载)，需要加载一次cache line，loop2每次循环正好跨16个元素，也就是每次循环需要加载一次cache line, 而loop3因为循环的步长为1024，虽然每次也要访问cache line但是总共的加载cache line的次数远低于loop1和loop2