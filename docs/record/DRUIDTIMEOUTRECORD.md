### alibaba druid抛出连接异常

    -连接池最大连接数为20，log中出现获取连接超时，代表20个连接都在占用
    -同时出现socket超时，服务中有一个动作是http调用第三方接口，调用接口时超时
    因为有事物的存在，调用第三方接口一直在等待超时，所以db的连接也就没有释放掉
    -请求数比db连接数多的时候，就浮现这个问题了
 #### 部分代码，简单的功能性测试不一定能测出来