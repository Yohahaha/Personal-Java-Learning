#### MongoDB-docker安装

`docker pull mongo`

`docker pull mongo-express`

新建`docker-compose-mongo.yml`文件

```yml
version: '3'
services:
  mongo:
    image: mongo:latest
    container_name: mongo
    volumes:
      - /mydata/mongo/db:/data/db #数据文件挂载
    ports:
      - 27017:27017
  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    ports:
      - 8081:8081
```

启动mongo

```bash
docker-compose -f docker-compose-mongo.yml up -d
```

> -f 指定启动文件
>
> -d 后台启动

这样就启动了一个基本的mongo服务以及web管理界面

访问host:8081即可看到mongo express管理界面

#### 其他配置

可以配置访问mongo的用户和密码

可以配置初始db，如果需要在初始化数据库时新增数据，写一个js文件可以在创建时导入数据