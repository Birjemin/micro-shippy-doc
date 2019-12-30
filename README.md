## go-micro微服务学习-货运系统
因为众所周知的原因，一直在探寻Golang在业务上的锲合度，所以花了一定的时间简单的搭建
了两个适合自己业务的模板（参见参考1、参考2），这两个模板分别采用了
[iris](http://github.com/kataras/iris)框架和
[gin](http://github.com/gin-gonic/gin)框架，两个模板都包含了平滑关闭、
database连接池和redis连接池、GRPC调用（欢迎食用，如果有其他意见也欢迎request）。
在此过程中，偶然看到了一篇关于go-micro教程（参考3），觉得十分有趣，并且一步步的
实践完成，在此记录一下整个过程，如果有小伙伴对这个go-micro感兴趣，对微服务感兴趣，
可以参考一下下，希望可以帮助到各位。(PS:我在社区看到有人翻译成中文了，感兴趣也可以看译文。)

## 示意图

整个微服务的示意图如下：

![示意图](./img/2019122800.png)

使用到的技术栈：
Docker、MongoDB、go-micro、grpc、protobuf、NATS、JWT、Postgres、Kubernetes...

注意：
完整代码见github仓库，文章中的代码只会例举一些关键性的改动，方便阅读。

## 目录
第一部分：建立consignment-cli和consignment-service服务
第二部分：完善第一部分，添加GetConsignments方法
第三部分：引入docker部署服务

## 参考
1. [https://github.com/Birjemin/gin-structure](https://github.com/Birjemin/gin-structure)
2. [https://github.com/Birjemin/iris-structure](https://github.com/Birjemin/iris-structure)
3. [https://ewanvalentine.io/microservices-in-golang-part-1/](https://ewanvalentine.io/microservices-in-golang-part-1/)