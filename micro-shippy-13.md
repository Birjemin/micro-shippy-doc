## 第十三部分：开启api网关

### 开始
```
go get github.com/gin-gonic/gin
```

#### 修改docker-compose.yml

```
...
  api:
    build: ./api
    ports:
      - "8180:8080"
    environment:
      MICRO_ADDRESS: ":8080"
...
```

#### 修改user-service服务
repository.go
```
...
func (repo *UserRepository) Get(id string) (*pb.User, error) {
    user := &pb.User{Id: id}
    if err := repo.db.First(&user).Error; err != nil {
        return nil, err
    }
    return user, nil
}
...
```

#### 修改vessel-service服务
main.go文件中注释掉createDummyData方法


#### 修改docker-compose.yml

#### 测试

database窗口，开启web服务
```
docker-compose up
docker-compose ps
```
![2019122872.png](img/2019122872.png)
1. 创建用户
![2019122863.png](img/2019122863.png)
2. 查看Postgres看users表单数据
![2019122864.png](img/2019122864.png)
3. 获取用户信息和token信息
![2019122867.png](img/2019122867.png)
4. 发起货运请求（没有货船~所以失败啦）
![2019122868.png](img/2019122868.png)
5. 创建货船
![2019122865.png](img/2019122865.png)
6. 查看MongoDB中的数据
![2019122866.png](img/2019122866.png)
7. 发起货运请求
![2019122869.png](img/2019122869.png)
