## 第四节：从grpc到go-micro

### 准备工作
- 什么是[go-micro](https://github.com/micro/go-micro)
- 引入go-micro
```
go get github.com/micro/go-micro
```

### 开始
go-micro是一个微服务框架，从这一节开始正式进入正题，将代码从grpc切换到go-micro框架上，如果对go-micro不太了解，或者对go-micro和grpc之间的关系有何区别，可以去官网查看一下它的架构和一些基本的概念。

#### 修改consignment-service

##### 修改Makefile
使用的插件是micro
```
build:
	protoc -I. --go_out=plugins=micro:. \
	  proto/consignment/consignment.proto
	GOOS=linux GOARCH=amd64 go build
	docker build -t consignment-service .
run:
	docker run -p 50051:50051 \
	    -e MICRO_SERVER_ADDRESS=:50051 \
	    consignment-service
```

##### 修改main.go中的代码

```
...
func main() {
    repo := &Repository{}

    // 创建服务
    srv := micro.NewService(
        // This name must match the package name given in your protobuf definition
        micro.Name("go.micro.srv.consignment"),
    )

    // 处理化
    srv.Init()

    // 注册Handler
    pb.RegisterShippingServiceHandler(srv.Server(), &service{repo})

    // 运行服务
    if err := srv.Run(); err != nil {
        fmt.Println(err)
    }
}
```
#### 修改consignment-cli
##### 添加Makefile

```
build:
	GOOS=linux GOARCH=amd64 go build
	docker build -t consignment-cli .
run:
	docker run consignment-cli
```

##### 添加Dockerfile

```
FROM alpine:latest

RUN mkdir -p /app
WORKDIR /app

ADD consignment.json /app/consignment.json
ADD consignment-cli /app/consignment-cli

CMD ["./consignment-cli"]
```

##### 修改cli.go文件
```
...
func parseFile(file string) (*pb.Consignment, error) {
    ...
}

func main() {
    service := micro.NewService(micro.Name("go.micro.cli.consignment"))
    service.Init()

    client := pb.NewShippingServiceClient("go.micro.srv.consignment", service.Client())

    // 可忽略这几行代码
    file := defaultFilename
    if len(os.Args) > 1 {
        file = os.Args[1]
    }

    consignment, err := parseFile(file)

    if err != nil {
        log.Fatalf("Could not parse file: %v", err)
    }

    r, err := client.CreateConsignment(context.Background(), consignment)
    if err != nil {
        log.Fatalf("Could not greet: %v", err)
    }
    log.Printf("Created: %t", r.Created)

    getAll, err := client.GetConsignments(context.Background(), &pb.GetRequest{})
    if err != nil {
        log.Fatalf("Could not list consignments: %v", err)
    }
    for _, v := range getAll.Consignments {
        log.Println(v)
    }
}

```

#### 测试
分别在两个窗口执行下面命令（会自动拉取依赖）

```
// 构建
make build
// 运行
make run
```

![2019122809.png](./img/2019122809.png)

![2019122810.png](./img/2019122810.png)

#### 当前的文件目录
```
$GOPATH/src
    └── micro-shippy
        ├── README.md
        ├── consignment-cli
        │   ├── Dockerfile
        │   ├── Makefile
        │   ├── cli.go
        │   ├── consignment-cli
        │   └── consignment.json
        ├── consignment-service
        │   ├── Dockerfile
        │   ├── Makefile
        │   ├── consignment-service
        │   ├── main.go
        │   └── proto
        │       └── consignment
        │           ├── consignment.pb.go
        │           └── consignment.proto
        ├── go.mod
        └── go.sum

```