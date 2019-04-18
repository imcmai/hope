##异构数据源高效同步datax
按照客户不同维度的条件来查询客户信息的需求，客户维度不同的数据都是加工落地在不同的数据源的。
为了更高的查询效率，从不同的数据源使用datax聚合数据同步落地到项目中的数据源，方便创建索引和减少查询复杂度提升查询效率。
github repo:https://github.com/alibaba/DataX
###简介
DataX 是阿里巴巴集团内被广泛使用的离线数据同步工具/平台，
实现包括 MySQL、Oracle、SqlServer、Postgre、HDFS、Hive、ADS、HBase、TableStore(OTS)、MaxCompute(ODPS)、DRDS 等各种异构数据源之间高效的数据同步功能。
####主要优势
- 脏数据检测
    在大量数据的传输过程中，必定会由于各种原因导致很多数据传输报错(比如类型转换错误)，这种数据DataX认为就是脏数据。DataX目前可以实现脏数据精确过滤、识别、采集、展示，为用户提供多种的脏数据处理模式
- 流量、数据量监控
    DataX3.0运行过程中可以将作业本身状态、数据流量、数据速度、执行进度等信息进行全面的展示，让用户可以实时了解作业状态。并可在作业执行过程中智能判断源端和目的端的速度对比情况，给予用户更多性能排查信息
- 速度控制
    DataX3.0提供了包括通道(并发)、记录流、字节流三种流控模式，可以随意控制你的作业速度
- 强劲的同步性能
    根据切分策略,将作业合理切分成多个Task并行执行,多线程执行模型可以让DataX速度随并发成线性增长
###环境
基于python，需要部署python2.7.x
###使用
####配置文件示例
执行datax需要配置datax可识别的json文件
json主要分为reader和writer，对应读取和写入的数据源
下面贴两个oracle->mysql和hbase->mysql
```java
{ 
 
    "job": { 
 
        "content": [ 
 
            { 
		        //读取数据源配置
                "reader": { 
		            //数据源:详参见datax Support Data Channels 如:oraclereader
                    "name": "oraclereader",
 
                    "parameter": { 
			            //连接用户名
                        "username": "test", 
			            //连接密码
                        "password": "test",  
                        //可用于增量更新
                        "where":"",
			            //需要同步的列
			            "column": [
                            "id","name"
                        ],
                        "connection": [ 
                            { 
				                //需要同步的数据，通过querysql来指定，可以在同步时就做好sql级的加工
                                "querySql": [ 
                                    select name uname from test
                                ], 
                                //也可以指定表名
                                "table":[
                                    test
                                ],
                                //jdbc连接地址
                                "jdbcUrl": [ 
                                    "jdbc:oracle:thin:@//***.***.*.*:**/test"
                                ] 
                            } 
                        ] 
                    } 
                }, 
		        //写数据源配置
                "writer": { 
                    "name": "mysqlwriter",
                    "parameter": { 
                        "username": "", 
                        "password": "", 
                        "column": [],
                        //执行写操作前的语句
                        "preSql":[
                            "delete from test"
                        ],
                        "connection": [ 
                            { 
                                "table": [ 
                                    "test" 
                                ], 
                                "jdbcUrl":"jdbc:mysql://ip:port/test?useUnicode=true&characterEncoding=GBK&autoReconnect=true&failOverReadOnly=false&tinyInt1isBit=false"                              
                            } 
                        ]   
                    } 
                } 
            } 
        ], 
        "setting": {
		        //效率控制
                 "speed": {
                          //设置传输速度 byte/s 尽量逼近这个速度但是不高于它.
                        // channel 表示通道数量，byte表示通道速度，如果单通道速度1MB，配置byte为1048576表示一个channel
                         "channel": 1,
                         "byte": 104857600
                 },
		            //出错限制
                 "errorLimit": {
			            //脏数据条数，优先选择record
                         "record": 10,
			            //脏数据百分比
                         "percentage": 0.05
		}
	}
    } 
}
```
```java
{ 
 
    "job": { 
 
        "content": [ 
 
            { 
 
                "reader": { 
 
                    "name": "hbase11xreader",
 
                    "parameter": { 
                        "hbaseConfig":	{
                        "hbase.zookeeper.quorum":"ip,ip,ip",
                        "hbase.zookeeper.property.clientPort":"port",
                        "zookeeper.znode.parent":"/hbase",
                        "hbase.client.retries.number":"1",
                        "hbase.rpc.timeout":"5000"
			         }, 
 
                        "table": "user:info",  
						
                        "encoding": "utf-8", 
                        //normal模式读取，把hbase作为二维表来读取	
                        "mode":	"normal",
                        //normal 模式下：name指定读取的hbase列，除了rowkey外，必须为 列族:列名 的格式，type指定源数据的类型，format指定日期类型的格式，value指定当前类型为常量，不从hbase读取数据，而是根据value值自动生成对应的列			
                        "column": [ 
                            {
                                "name":"rowkey",
                                "type":"string"
                            },
                            {
                                "name":"info:name",
                                "type":"string"
                            },
                            {
                                "name":"info:id",
                                "type":"string"
                            },
                            {
                                "name":"info:phone",
                                "type":"string"
                            },
                            {
                                "name":"info:email",
                                "type":"string"
                            }
                        ],
                        "range":{
                            //指定起始rowkey
                            "startRowkey":"",
                            //指定结束rowkey
                            "endRowKey":"",
                            //指定配置的startRowkey和endRowkey转换为byte[]时的方式，默认值为false,若为true，则调用Bytes.toBytesBinary(rowkey)方法进行转换;若为false：则调用Bytes.toBytes(rowkey)
                            "isBinaryRowKey":true
                        }
                    } 
                }, 
                "writer": { 
                    "name": "mysqlwriter",
                    "parameter": { 
                        "username": "", 
                        "password": "", 
                        "column": ["","","",""],
                        //同步完成后执行
                        "postSql":[
                            ""
                        ],
                        "preSql":[
                            ""
                        ],
                        "connection": [ 
                            { 
                                "table": [ 
                                    "" 
                                ], 
                                "jdbcUrl":"jdbc:mysql://ip:port/test?useUnicode=true&characterEncoding=GBK&autoReconnect=true&failOverReadOnly=false&tinyInt1isBit=false&socketTimeout=300000"                              
                            } 
                        ]   
                    } 
                } 
            } 
        ], 
        "setting": {
                 "speed": {
                         "channel": 1,
                         "byte": 104857600
                 },
                 "errorLimit": {
                         "record": 10,
                         "percentage": 0.05
		        }
	    }
    } 
}

```
####执行
1.windows直接cmd执行datax的核心文件datax.py并指定需要解析的配置文件即可，如:python path/bin/datax.py path/dataxconfig/dataxtest.json
2.shell脚本执行
3.java调用Runtime.getRuntime().exec(param)通知JVM执行服务器命令，执行时为阻塞状态
```java
    String[] params = new String[]{"python","dataxpath","configpath"};
    Runtime.getRuntime().exec(params);
```
我在此次项目用的就是这一种，方便在datax同步前后做一些逻辑处理
####datax入口
datax入口函数 datax.py __main__
```java
printCopyright()
parser = getOptionParser(sys.argv[1:])
options, args = parser.parse_args(sys.argv[1:])
if options.reader is not None and options.writer is not None:
    generateJobConfigTemplate(options.reader,options.writer)
    sys.exit(RET_STATE['OK'])
if len(args) == 0:
    parser.print_help()
    sys.exit(RET_STATE['FAIL'])
startCommand = buildStartCommand(options, args)
child_process = subprocess.Popen(startCommand, shell=True)
```
大意就是拼接参数启动新的进程去执行java命令
####提示
部署datax时可删除无用plugin，否则datax整体软件包会很膨胀
好了，去体验datax的极速吧
