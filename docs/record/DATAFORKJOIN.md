# 背景
### 线上笔试题
**随机生成 Salary {name, baseSalary, bonus  }的记录，如“wxxx,10,1”，每行一条记录，总共1000万记录，写入文本文件（UFT-8编码），然后读取文件，name的前两个字符相同的，其年薪累加，比如wx，100万，3个人，最后做排序和分组，输出年薪总额最高的10组,并将其尽量优化到五秒**
```java
package subject.demo.two;

import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.File;
import java.io.FileReader;
import java.io.FileWriter;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Random;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
public class Application {

    static int append = 0;

    public static void main(String[] args) throws IOException, InterruptedException {
        //文件
        File file = new File("D:/demo2.txt");
        if (file.exists()) {
            file.delete();
            file.createNewFile();
        }
        FileWriter fw = new FileWriter(file.getAbsoluteFile(), true);
        BufferedWriter bw = new BufferedWriter(fw);

        long startTime = System.currentTimeMillis();

        //记录条数
        final int count = 10000000;
        final int batch = 1000;
        final int lop = count/batch;
        final CountDownLatch countDown = new CountDownLatch(lop);

        ThreadPoolExecutor service = new ThreadPoolExecutor(1000, 1000, 0, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<Runnable>());
        for(int start = 0; start < lop; start ++) {
            service.execute(() -> {
                StringBuffer stringBuffer = new StringBuffer();
                for (int i=0; i < batch; i++){
                    stringBuffer.append(getRandomString(4) + "," +(int) (Math.random() * 9000 + 1000) + "," +(int) (Math.random() * 9000 + 1000));
                    stringBuffer.append("\r\n");
                }
                try {
                    synchronized (bw) {
                        bw.write(stringBuffer.toString());
                        bw.flush();
                        countDown.countDown();
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            });
        }
        countDown.await();
        bw.close();
        System.out.println("insert finish");

        FileReader reader = new FileReader(file);
        BufferedReader br = new BufferedReader(reader);
        String line = null;
        int c = 0;

        Map<String, Integer> map = new HashMap<>();
        Map<String,Long> countMap = new HashMap<>();

        int lineCount = 0;
        while((line = br.readLine()) != null) {
            final String[] infos = line.split("[,]");
            String start = infos[0].substring(0, 2);
            int yearSalary = Integer.parseInt(infos[1]) + Integer.parseInt(infos[2]);
            Integer s = map.get(start);
            if(s != null) {
                map.put(start, yearSalary + s);
                countMap.put(start,countMap.get(start)+s);
            }else {
                map.put(start, yearSalary);
                countMap.put(start,1L);
            }
            ++lineCount;
        }
        List<Map.Entry<String,Integer>> list = new ArrayList<Map.Entry<String,Integer>>(map.entrySet());
        Collections.sort(list, (o1, o2) -> o1.getValue().compareTo(o2.getValue()));
        for (int i =0;i<10;i++){
            String key = list.get(i).getKey();
            System.out.println(key+","+list.get(i).getValue()+"万,"+countMap.get(key)+"人");
        }
        System.out.println("line：" + lineCount);
        System.out.println("耗时:" + (System.currentTimeMillis() - startTime) + " ms");
        service.shutdown();
    }

    public static String getRandomString(int length){
        String str="abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
        Random random=new Random();
        StringBuffer sb=new StringBuffer();
        for(int i=0;i<length;i++){
            int number=random.nextInt(5);
            sb.append(str.charAt(number));
        }
        return sb.toString();
    }
}

```
***
其实还是有点问题，当时比较仓促，其实分组和排序可以在插入的时候做，不用再读取一遍
forkjoin思路来自于表哥