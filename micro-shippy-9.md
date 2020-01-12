# 第九节：引入jwt鉴权

## 准备工作
- 什么是jwt鉴权？

## 开始
引入jwt的golang库：
```
go get github.com/dgrijalva/jwt-go
```
该节代码量有点多，调试也有点麻烦。

### user-service服务
为了引入jwt鉴权，需要将第八节的用户服务完善。

#### 修改handler.go
完善授权和鉴权的方法。
```
...
func (srv *service) Auth(ctx context.Context, req *pb.User, res *pb.Token) error {
    log.Println("Logging in with:", req.Email, req.Password)
    user, err := srv.repo.GetByEmail(req.Email)
    log.Println(user)
    if err != nil {
        return err
    }

    if err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(req.Password)); err != nil {
        return err
    }

    token, err := srv.tokenService.Encode(user)
    if err != nil {
        return err
    }
    res.Token = token
    return nil
}
...
func (srv *service) ValidateToken(ctx context.Context, req *pb.Token, res *pb.Token) error {

    if req.Token == "" {
        return errors.New("token invalid")
    }
    // 解密
    claims, err := srv.tokenService.Decode(req.Token)
    if err != nil {
        return err
    }

    log.Println(claims)

    if claims.User.Id == "" {
        return errors.New("invalid user")
    }

    res.Valid = true

    return nil
}
```

#### 修改repository.go文件
```
...
func (repo *UserRepository) GetByEmail(email string) (*pb.User, error) {
    user := &pb.User{}
    if err := repo.db.Where("email = ?", email).
        First(&user).Error; err != nil {
        return nil, err
    }
    return user, nil
}
...
```

#### 修改token_service.go文件
省略

### 修改user-cli访问终端
#### 修改cli.go
```
package main

import (
    pb "github.com/birjemin/micro-shippy/user-service/proto/user"
    microclient "github.com/micro/go-micro/client"
    "github.com/micro/go-micro/config/cmd"
    "golang.org/x/net/context"
    "log"
    "os"
)

func main() {

    cmd.Init()

    client := pb.NewUserServiceClient("go.micro.srv.user", microclient.DefaultClient)
    // 先直接写死一个用户
    name := "Ewan Valentine"
    email := "ewan.valentine89@gmail.com"
    password := "test123"
    company := "BBC"

    log.Println(name, email, password)

    r, err := client.Create(context.TODO(), &pb.User{
        Name:     name,
        Email:    email,
        Password: password,
        Company:  company,
    })
    if err != nil {
        log.Fatalf("Could not create user: %v", err)
    }
    log.Printf("Created: %s", r.User.Id)

    getAll, err := client.GetAll(context.Background(), &pb.Request{})
    if err != nil {
        log.Fatalf("Could not list users: %v", err)
    }
    for _, v := range getAll.Users {
        log.Println(v)
    }

    authResponse, err := client.Auth(context.TODO(), &pb.User{
        Email:    email,
        Password: password,
    })

    if err != nil {
        log.Fatalf("Could not authenticate user: %s error: %v\n", email, err)
    }

    log.Printf("Your access token is: %s \n", authResponse.Token)
    os.Exit(0)
}
```

### 修改consignment-service
给托运服务增加授权功能。

#### 修改main.go
```
...

// 授权中间件
func AuthWrapper(fn server.HandlerFunc) server.HandlerFunc {
    return func(ctx context.Context, req server.Request, resp interface{}) error {
        meta, ok := metadata.FromContext(ctx)
        if !ok {
            return errors.New("no auth meta-data found in request")
        }
        token := meta["Token"]
        log.Println("Authenticating with token: ", token)

        // 授权
        authClient := userService.NewUserServiceClient("go.micro.srv.user", client.DefaultClient)
        authResp, err := authClient.ValidateToken(ctx, &userService.Token{
            Token: token,
        })
        log.Println("Auth resp:", authResp)
        log.Println("Err:", err)
        if err != nil {
            return err
        }

        err = fn(ctx, req, resp)
        return err
    }
}
```

### 修改consignment-cli
托运cli增加token入口，避免将token写死，无法调试。

#### 修改cli.go
```
...
    ctx := metadata.NewContext(context.Background(), map[string]string{
        "token": token,
    })

    r, err := client.CreateConsignment(ctx, consignment)
    if err != nil {
        log.Fatalf("Could not create: %v", err)
    }
    log.Printf("Created: %t", r.Created)

    getAll, err := client.GetConsignments(ctx, &pb.GetRequest{})
    if err != nil {
        log.Fatalf("Could not list consignments: %v", err)
    }
    for _, v := range getAll.Consignments {
        log.Println(v)
    }
...
```

### 测试

database窗口
```
docker-compose ps
```
![2019122830.png](./img/2019122830.png)

vessel-service开启：

```
docker-compose run vessel-service 
```
![2019122841.png](img/2019122841.png)

consignment-service窗口：

```
make build
docker-compose build --no-cache consignment-service 
docker-compose run consignment-service 
```
![2019122840.png](img/2019122840.png)

user-service窗口：

```
make build
docker-compose build --no-cache user-service 
docker-compose run user-service 
```
![2019122835.png](img/2019122835.png)

user-cli窗口：

```
make build
docker-compose build --no-cache user-cli 
docker-compose run user-cli 
```
![2019122834.png](./img/2019122834.png)
得到TOKEN的值！！

此时user-service窗口：
![2019122836.png](./img/2019122836.png)

Postgres数据新增一条：
![2019122837.png](./img/2019122837.png)

consignment-cli窗口：

```
TOKEN=xxx docker-compose run user-service 
```
![2019122838.png](./img/2019122838.png)

此时consignment-service窗口变化：
![2019122839.png](./img/2019122839.png)

此时MongoDB中会生成一条数据（货运数据）：
![2019122842.png](./img/2019122842.png)

### 当前的文件目录
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