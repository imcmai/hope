# xss注入
[xss测试网站](http://xss.fbisb.com/)
## level1
![xss1](../_media/xss1.png)
第一题很简单，用户的输入会从url发到服务器后直接转到页面，如图
![xss1.1](../_media/xss1.1.png)
所以直接从url注入scrpit脚本
![xss1.2](../_media/xss1.2.png)
ok,顺利通关
![xss1.3](../_media/xss1.3.png)
## level2
url为:
https://xss.fbisb.com/yx/level2.php?keyword=test 

keyword会回显到页面input的value属性，url改为
https://xss.fbisb.com/yx/level2.php?keyword=%22%3E%3Cscript%3Ealert(%22level2%22)%3C/script%3E,闭合原有input标签，插入script脚本后通关
