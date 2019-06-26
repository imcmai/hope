## arthas(阿尔萨斯)是什么
arthas是阿里巴巴开源的java诊断工具，仅支持JDK6+(arthas使用了Java Agent，只有JDK6+才支持的进程间通讯使用Java Agent)
### 官方描述
Arthas 是一款线上监控诊断产品，通过全局视角实时查看应用 load、内存、gc、线程的状态信息，并能在不修改应用代码的情况下，
对业务问题进行诊断，包括查看方法调用的出入参、异常，监测方法执行耗时，类加载信息等，大大提升线上问题排查效率。
[在线教程-可以直接使用在线服务器](https://alibaba.github.io/arthas/arthas-tutorials?language=cn)
## 快速开始
### 安装
下载arthas-boot
```code
wget https://alibaba.github.io/arthas/arthas-boot.jar
java -jar arthas-boot.jar
```
或者
```java
curl -L https://alibaba.github.io/arthas/install.sh | sh
./as.sh
```
第二种启动需要服务器安装telnet
### 注
启动时arthas会使用jps -lv来列出所有的java进程，选择一个进程就可以使其开始监控这个进程并且进入命令交互



