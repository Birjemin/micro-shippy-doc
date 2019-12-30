## 第十二部分：开启web端交互

### 开始

#### 修改docker-compose.yml

```
version: '3.1'
...
    web:
      command: web
      image: micro/micro:latest
      ports:
        - "8182:8082"
      environment:
        MICRO_ADDRESS: ":8082"
...
```

#### 测试

database窗口，开启web服务
```
docker-compose up --no-start web
docker-compose start web 
docker-compose ps
```
![2019122855.png](./img/2019122855.png)

访问localhost:8182开启：
![2019122856.png](img/2019122856.png)

![2019122857.png](img/2019122857.png)
![2019122858.png](img/2019122858.png)
![2019122859.png](img/2019122859.png)

user-service窗口变化：
![2019122860.png](./img/2019122860.png)

email-service窗口变化：
![2019122861.png](./img/2019122861.png)

数据库数据变化：
![2019122862.png](img/2019122862.png)