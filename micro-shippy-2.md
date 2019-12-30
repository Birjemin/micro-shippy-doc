## 第二部分：添加GetConsignments方法


### 开始

#### 修改consignment-service服务

##### 修改protobuf通信协议
在consignment.proto修改成下面内容：
```
syntax = "proto3";

package consignment;

// 货轮服务
service ShippingService {
    // 托运货物
    rpc CreateConsignment(Consignment) returns (Response) {}
    // 添加查看托运货物方法
    rpc GetConsignments(GetRequest) returns (Response) {}
}
// 货物属性（id、描述、重量、包含的集装箱、货船id）
message Consignment {
    string id = 1;
    string description = 2;
    int32 weight = 3;
    repeated Container containers = 4;
    string vessel_id = 5;
}
// 单个集装箱（id、客户id、来源、用户id）
message Container {
    string id = 1;
    string customer_id = 2;
    string origin = 3;
    string user_id = 4;
}

// 查询请求
message GetRequest {}

// 托运结果（托运结果，托运货物，目前托运的货物）
message Response {
    bool created = 1;
    Consignment consignment = 2;
    repeated Consignment consignments = 3;
}
```

##### 生成协议代码

执行命令：

```sh
make build
```
在consignment目录中会重新生成`consignment.pb.go`文件

##### 修改consignment-service服务

在main.go文件中新增GetAll方法：

```
...
type repository interface {
    Create(*pb.Consignment) (*pb.Consignment, error)
    // 新增GetAll方法
    GetAll() []*pb.Consignment
}

// Repository - Dummy repository, this simulates the use of a datastore
// of some kind. We'll replace this with a real implementation later on.
type Repository struct {
    mu           sync.RWMutex
    consignments []*pb.Consignment
}

// Create a new consignment
func (repo *Repository) Create(consignment *pb.Consignment) (*pb.Consignment, error) {
    ...
}

// GetAll consignments
func (repo *Repository) GetAll() []*pb.Consignment {
    return repo.consignments
}

// Service should implement all of the methods to satisfy the service
// we defined in our protobuf definition. You can check the interface
// in the generated code itself for the exact method signatures etc
// to give you a better idea.
type service struct {
    repo repository
}

// CreateConsignment - we created just one method on our service,
// which is a create method, which takes a context and a request as an
// argument, these are handled by the gRPC server.
func (s *service) CreateConsignment(ctx context.Context, req *pb.Consignment) (*pb.Response, error) {
    ...
}

// 新增GetConsignments方法
func (s *service) GetConsignments(ctx context.Context, req *pb.GetRequest) (*pb.Response, error) {
    consignments := s.repo.GetAll()
    return &pb.Response{Consignments: consignments}, nil
}

func main() {

    ...
}

```

#### 修改consignment-cli访问终端
##### 目录结构：

##### 修改cli.go

```
func parseFile(file string) (*pb.Consignment, error) {
    ...
}

func main() {
    // Set up a connection to the server.
    ...

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
分别在连个窗口执行下面命令（会自动拉取依赖）

```
go run main.go
```

![2019122805.png](./img/2019122805.png)

```
go run cli.go
```
![2019122806.png](./img/2019122806.png)

#### 当前的文件目录
```
$GOPATH/src
    └── micro-shippy
        ├── README.md
        ├── consignment-cli
        │   ├── cli.go
        │   └── consignment.json
        ├── consignment-service
        │   ├── Makefile
        │   ├── main.go
        │   └── proto
        │       └── consignment
        │           ├── consignment.pb.go
        │           └── consignment.proto
        ├── go.mod
        └── go.sum
```