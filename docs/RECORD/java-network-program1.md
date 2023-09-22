## 前言
不作为生产环境使用，仅用于学习目的
在使用mina、netty等网络编程框架之前，我们先来聊一下socket、nio这些知识
## socket是什么？
简单来讲scoket定义了一些抽象的API，位于应用层和传输层之间，用作计算机网络中的通信，实现上是一个双向的通信机制，底层使用TCP或者UDP的协议
![socket](../_media/socket_1.png)
#### 内存溢出
## 基于TCP的socket基本工作流程
![socket2](../_media/socket_2.jpeg)

**服务端**
1. socket(),创建socket实例
2. bind(),绑定ip 端口到socket实例
3. listen(),指定监听端口，可以是多个，通常和bind一起使用，最终监听的端口由listen()决定
4. accept(),阻塞等待连接进来，阻塞机制依赖于操作系统的底层实现，类似于等待、唤醒的机制，防止过度消耗cpu资源
5. read()，处理请求,也是一个阻塞函数
6. write(),回写响应

**客户端**
1. socket(),创建socket实例
2. connect()，指定本地端口和远程的ip、端口，创建连接
3. write()，写入请求
4. read()，处理响应

## socket怎么用？
以下内容以基于TCP的socket作为示例  

**服务端**
```
        // 创建ServerSocket对象，并绑定到指定端口
        ServerSocket serverSocket = new ServerSocket(8080);

        System.out.println("服务器已启动，等待客户端连接...");

        // 接受客户端连接请求
        Socket clientSocket = serverSocket.accept();

        System.out.println("客户端已连接，IP地址为：" + clientSocket.getInetAddress());

        // 获取输入流和输出流
        BufferedReader input = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
        PrintWriter output = new PrintWriter(clientSocket.getOutputStream(), true);

        // 从客户端接收消息
        String message = input.readLine();
        System.out.println("客户端的消息：" + message);

        // 向客户端发送消息
        output.println("Hello, Client!");

        // 关闭连接
        clientSocket.close();
        serverSocket.close();
```
**客户端**
```
        // 创建Socket对象，指定服务器的IP地址和端口号
        Socket socket = new Socket("localhost", 8080);

        // 获取输入流和输出流
        BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
        PrintWriter output = new PrintWriter(socket.getOutputStream(), true);

        // 向服务器发送消息
        output.println("Hello, Server!");

        // 从服务器接收消息
        String response = input.readLine();
        System.out.println("服务器的响应：" + response);
        // 关闭连接
        socket.close();
```
这是一个简单socket实例，服务端创建了一个socket实例，并且监听了8080端口，通过accept()方法等待客户端连接，客户端则是制定了服务器的ip和端口，创建了连接，通过输出流和输入流发送和接收消息。  
这样会有一个问题就是<font color = #FF000 size=3 >服务端只能接收一次请求进程就关闭了</font>，我们需要处理的是多个请求,那么稍微改造一下服务端代码,给外层套上一个无限循环，这样我们就可以持续监听端口来接收客户端的连接了，同时这也是这是<font color = #FF000 size=3 >服务端编程一个常见的编程模型</font>
```
        while (true) {
            // 接受客户端连接请求
            Socket clientSocket = serverSocket.accept();
        }
```
然后我们就产生了新的问题，这种循环只能串行去一个一个的处理请求，所以我们还需要分配线程来应对并发的问题，我们对服务端的代码再改造一下
```
        //无界队列，核心线程数和最大线程数都为10(根据实际情况调整)
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        // 创建ServerSocket对象，并绑定到指定端口
        ServerSocket serverSocket = new ServerSocket(8080);

        System.out.println("服务器已启动，等待客户端连接...");
        while (true) {
            // 接受客户端连接请求
            Socket clientSocket = serverSocket.accept();
            System.out.println("客户端已连接，IP地址为：" + clientSocket.getInetAddress());
            executorService.execute(() -> {
                try {
                    // 获取输入流和输出流
                    BufferedReader input = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
                    PrintWriter output = new PrintWriter(clientSocket.getOutputStream(), true);

                    // 从客户端接收消息
                    String message = input.readLine();
                    System.out.println("客户端的消息：" + message);

                    // 向客户端发送消息
                    output.println("Hello, Client!");

                    // 关闭连接
                    clientSocket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            });
        }
```
好，处理请求这里我们暂时告一段落，接下来看看序列化这里，真实的网络通信过程中，消息的内容会更复杂，很多中间件都会定义自己的协议
> 协议是什么？，协议是约定通信格式，最关键的就是确保服务端和客户端能够正确的序列化和反序列化消息  
### RESP协议
我们先来看看redis定义的RESP协议,下面以一个简单的auth和set请求举例
```code
        try (Socket socket = new Socket("redis", 6379);
             OutputStream outputStream = socket.getOutputStream();
             InputStream inputStream = socket.getInputStream();
             BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream))) {

            // 身份验证
            String password = "095a7a7b8de942a88e6a440fde8d53b2";
            String authCommand = "*2\r\n$4\r\nAUTH\r\n$" + password.length() + "\r\n" + password + "\r\n";
            outputStream.write(authCommand.getBytes());

            // 读取Redis服务器的响应
            String response = reader.readLine();
            System.out.println("Response from Redis: " + response);

            // 构建RESP协议的请求
            String[] command = new String[]{"SET", "mykey", "Hello Redis,Hello RESP!"};
            StringBuilder sb = new StringBuilder();
            sb.append("*").append(command.length).append("\r\n");
            for (String arg : command) {
                sb.append("$").append(arg.getBytes().length).append("\r\n");
                sb.append(arg).append("\r\n");
            }
            String request = sb.toString();

            // 发送请求给Redis服务器
            outputStream.write(request.getBytes());
            outputStream.flush();

            // 解析RESP协议的响应
            String line;
            while ((line = reader.readLine()) != null) {
                System.out.println("Response from Redis:" + line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

```
>> Redis的RESP（REdis Serialization Protocol）协议是一种简单而高效的二进制协议，用于Redis客户端和服务器之间的通信。  

**RESP协议的编码和解码规则如下：**
 1. 每个请求或响应都以一个固定长度的数组开头，表示数组中元素的个数。使用 * 字符后跟一个数字，表示元素个数。例如，一个包含3个元素的数组编码为 *3\r\n。

 2. RESP协议支持多种数据类型，包括简单字符串、错误信息、整数、批量字符串和数组。每个数据类型都以一个特定的前缀字符开头，用于标识其类型。

    - 简单字符串：以 + 字符开头，后面跟着实际的字符串内容，以及一个 \r\n 作为结尾。例如，编码字符串 "OK" 的RESP表示为 +OK\r\n。
    
    - 错误信息：以 - 字符开头，后面跟着实际的错误消息，以及一个 \r\n 作为结尾。例如，编码错误消息 "Error" 的RESP表示为 -Error\r\n。
    
    - 整数：以 : 字符开头，后面跟着一个整数值的字符串表示形式，以及一个 \r\n 作为结尾。例如，编码整数 42 的RESP表示为 :42\r\n。
    
    - 批量字符串：以 $ 字符开头，后面跟着实际字符串的字节数量（使用字节数的字符串表示形式），然后是 \r\n。接下来是实际的字符串内容，以及一个 \r\n 作为结尾。例如，编码字符串 "Hello" 的RESP表示为 $5\r\nHello\r\n。

    - 数组：以 * 字符开头，后面跟着数组中元素的个数（使用数字的字符串表示形式），然后是 \r\n。接下来是数组中每个元素的编码表示，以及一个 \r\n 作为结尾。例如，编码数组 ["foo", "bar", "baz"] 的RESP表示为 *3\r\n$3\r\nfoo\r\n$3\r\nbar\r\n$3\r\nbaz\r\n。

简单字符串和批量字符串、错误信息通常用于表示服务器的响应，例如上面的client代码得到的就是
```
Response from Redis: +OK
```
请求基本用的都是数组形式，例如上面的auth请求，拼装后的完整指令为
```code
*2
$4
AUTH
$32
095a7a7b8de942a88e6a440fde8d53b2
```
*2表示元素中的个数，一个是密码长度，一个是密码，$4表示后面字符串的长度，auth是实际的命令字符串，$32是密码字符串的长度，095a7a7b8de942a88e6a440fde8d53b2则是具体的密码，中间换行的\r\n则是为了正确的切分和解析 
同理，下面的set指令拼接后的完整指令为
```
*3
$3
SET
$5
mykey
$23
```
怎么样，是不是很简洁，高效
### 开始我们的自定义协议
#### 协议结构
#### 编码
#### 解码