## 第十部分：引入Pubsub订阅

### 开始

#### user-service服务

##### 修改handler.go
```
...
type service struct {
    repo         Repository
    tokenService Authable
    Publisher    micro.Publisher
}
...
func (srv *service) Create(ctx context.Context, req *pb.User, res *pb.Response) error {
    ...
    // add publisher handle
    if err := srv.Publisher.Publish(ctx, req); err != nil {
        return err
    }
    return nil
}
...
```

##### main.go文件
```
...
    publisher := micro.NewPublisher("user.created", srv.Client())
    // Register handler
    pb.RegisterUserServiceHandler(srv.Server(), &service{repo, tokenService, publisher})
...
```

#### 增加email-service服务
##### 增加文件
见仓库

#### 修改docker-compose.yml

```
version: '3.1'
...
  email-service:
    build: ./email-service
...
```

#### 测试

database窗口
```
docker-compose up --no-start database
docker-compose start database 
docker-compose ps
```
![2019122830.png](./img/2019122830.png)

user-service开启：

```
make build
docker-compose build --no-cache user-service 
docker-compose run user-service 
```
![2019122843.png](img/2019122843.png)

email-service开启：

```
make build
docker-compose build --no-cache email-service 
docker-compose run email-service 
```
![2019122844.png](img/2019122844.png)

user-cli开启：

```
docker-compose run user-cli 
```
![2019122845.png](img/2019122845.png)

user-service窗口变化：
![2019122846.png](./img/2019122846.png)

email-service窗口变化：
![2019122847.png](./img/2019122847.png)

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
        │   ├── datastore.go
        │   ├── handler.go
        │   ├── main.go
        │   ├── proto
        │   │   └── consignment
        │   │       ├── consignment.pb.go
        │   │       └── consignment.proto
        │   └── repository.go
        ├── docker-compose.yml
        ├── email-service
        │   ├── Dockerfile
        │   ├── Makefile
        │   ├── email-service
        │   └── main.go
        ├── go.mod
        ├── go.sum
        ├── user-cli
        │   ├── Dockerfile
        │   ├── Makefile
        │   ├── cli.go
        │   └── user-cli
        ├── user-service
        │   ├── Dockerfile
        │   ├── Makefile
        │   ├── database.go
        │   ├── handler.go
        │   ├── main.go
        │   ├── proto
        │   │   └── user
        │   │       ├── extension.go
        │   │       ├── user.pb.go
        │   │       └── user.proto
        │   ├── repository.go
        │   ├── token_service.go
        │   └── user-service
        └── vessel-service
            ├── Dockerfile
            ├── Makefile
            ├── datastore.go
            ├── handler.go
            ├── main.go
            ├── proto
            │   └── vessel
            │       ├── vessel.pb.go
            │       └── vessel.proto
            ├── repository.go
            └── vessel-service


```