## 第五节：引入货轮服务

### 开始
这一节开始编写货轮服务，然后更改托运服务代码，将两者关联起来，这样就实现了托运服务和货轮服务之间的rpc调用。

#### 增加vessel-service
##### 目录结构：
```
$GOPATH/src
    └── micro-shippy
        └── vessel-service
            ├── Dockerfile
            ├── Makefile
            ├── main.go
            └── proto
                └── vessel
                    └── vessel.proto
```
##### 增加protobuf通信协议
在vessel.proto修改成下面内容，提供一个查找空闲货轮的方法：
```
syntax = "proto3";

package vessel;

service VesselService {
    // 查找空闲的货轮
    rpc FindAvailable(Specification) returns (Response) {}
}

// 货轮属性（id、可装载集装箱数量、最大重量、名称、是否可用、归属）
message Vessel {
    string id = 1;
    int32 capacity = 2;
    int32 max_weight = 3;
    string name = 4;
    bool available = 5;
    string owner_id = 6;
}

// 待运送的货物规格（集装箱数量，最大重量）
message Specification {
    int32 capacity = 1;
    int32 max_weight = 2;
}

// 返回（返回货轮信息）
message Response {
    Vessel vessel = 1;
    repeated Vessel vessels = 2;
}
```

##### 生成协议代码

执行命令：

```sh
make build
```
在vessel目录中会重新生成`vessel.pb.go`文件

##### 增加Makefile
```
build:
	protoc -I. --go_out=plugins=micro:. \
	  proto/vessel/vessel.proto
	GOOS=linux GOARCH=amd64 go build
	docker build -t vessel-service .
run:
	docker run -p 50052:50051 -e MICRO_SERVER_ADDRESS=:50051 vessel-service
```

##### 增加Dockerfile
```
FROM alpine:latest

RUN mkdir /app
WORKDIR /app
ADD vessel-service /app/vessel-service

CMD ["./vessel-service"]
```

##### 增加main.go代码
```
package main

import (
    "context"
    "errors"
    "fmt"
    pb "github.com/birjemin/micro-shippy/vessel-service/proto/vessel"
    "github.com/micro/go-micro"
)

type Repository interface {
    FindAvailable(*pb.Specification) (*pb.Vessel, error)
}

type VesselRepository struct {
    vessels []*pb.Vessel
}

// repository 查找可用的货轮
func (repo *VesselRepository) FindAvailable(spec *pb.Specification) (*pb.Vessel, error) {
    for _, vessel := range repo.vessels {
        if spec.Capacity <= vessel.Capacity && spec.MaxWeight <= vessel.MaxWeight {
            return vessel, nil
        }
    }
    return nil, errors.New("No vessel found by that spec")
}

type service struct {
    repo Repository
}

func (s *service) FindAvailable(ctx context.Context, req *pb.Specification, res *pb.Response) error {
    vessel, err := s.repo.FindAvailable(req)
    if err != nil {
        return err
    }

    // 返回可用的货轮
    res.Vessel = vessel
    return nil
}

func main() {
    // 初始化数据，服务开启时先默认增加一个可用的货轮
    vessels := []*pb.Vessel{
        {Id: "vessel001", Name: "Boaty McBoatface", MaxWeight: 200000, Capacity: 500},
    }
    repo := &VesselRepository{vessels}

    srv := micro.NewService(
        micro.Name("go.micro.srv.vessel"),
    )

    srv.Init()

    // 注册货轮服务
    pb.RegisterVesselServiceHandler(srv.Server(), &service{repo})

    if err := srv.Run(); err != nil {
        fmt.Println(err)
    }
}

```

#### 修改consignment-service
修改托运服务的代码，在托运服务中调用货轮服务

##### 修改main.go中的代码

```
...
type service struct {
    repo repository
    // 增加vesselClient
    vesselClient vesselProto.VesselServiceClient
}

func (s *service) CreateConsignment(ctx context.Context, req *pb.Consignment, res *pb.Response) error {
    // 调用货轮服务的FindAvailable方法，Capacity为集装箱数量
    vesselResponse, err := s.vesselClient.FindAvailable(context.Background(), &vesselProto.Specification{
        MaxWeight: req.Weight,
        Capacity: int32(len(req.Containers)),
    })
    log.Printf("Found vessel: %s \n", vesselResponse.Vessel.Name)
    if err != nil {
        return err
    }
    // 设置查找到的货轮Id
    req.VesselId = vesselResponse.Vessel.Id
    // 如果货轮可用，则创建托运
    consignment, err := s.repo.Create(req)
    if err != nil {
        return err
    }
    // 创建成功
    res.Created = true
    res.Consignment = consignment
    return nil
}

func (s *service) GetConsignments(ctx context.Context, req *pb.GetRequest, res *pb.Response) error {
    ...
}

func main() {

    repo := &Repository{}

    srv := micro.NewService(
        // This name must match the package name given in your protobuf definition
        micro.Name("go.micro.srv.consignment"),
    )

    srv.Init()
    // 引入货轮服务
    vesselClient := vesselProto.NewVesselServiceClient("go.micro.srv.vessel", srv.Client())

    // Register handlers
    pb.RegisterShippingServiceHandler(srv.Server(), &service{repo, vesselClient})

    // Run the server
    if err := srv.Run(); err != nil {
        fmt.Println(err)
    }
}

```
#### 修改consignment-cli
##### 修改consignment.json
修改托运cli的代码，增加重量和多个集装箱。
```
{
  "description": "This is a test consignment",
  "weight": 55000,
  "containers": [
    { "customer_id": "cust001", "user_id": "user001", "origin": "Manchester, United Kingdom" },
    { "customer_id": "cust002", "user_id": "user001", "origin": "Derby, United Kingdom" },
    { "customer_id": "cust005", "user_id": "user001", "origin": "Sheffield, United Kingdom" }
  ]
}
```

#### 测试
分别在三个窗口执行下面命令（会自动拉取依赖）

```
// 构建
make build
// 运行
make run
```
consignment-service窗口：
![2019122811.png](./img/2019122811.png)

vessel-service窗口：
![2019122812.png](./img/2019122812.png)

consignment-cli窗口：
![2019122813.png](./img/2019122813.png)

consignment-service窗口变化：
![2019122814.png](./img/2019122814.png)

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
        ├── go.sum
        └── vessel-service
            ├── Dockerfile
            ├── Makefile
            ├── main.go
            ├── proto
            │   └── vessel
            │       ├── vessel.pb.go
            │       └── vessel.proto
            └── vessel-service
```