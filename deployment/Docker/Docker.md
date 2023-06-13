# Docker



## 安装 Docker

> Windows 系统推荐安装 WSL2，然后再安装 Docker，安装完 Docker 记得修改镜像源为国内源

[阿里云服务器设置 Docker 镜像加速器](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)

[更改docker仓库源地址 1](https://blog.csdn.net/m0_37886429/article/details/80323149)

[更改docker仓库源地址 2](https://www.jianshu.com/p/df75f9b5fcf6)



**安装完成启动**

```shell
# ubuntu
service docker start
service docker restart
service docker stop

# centos 7
systemctl start docker
systemctl restart docker
systemctl stop docker
```



## 镜像启动



### MySQL 8

```shell
[root@localhost home]# docker run --name mysql01 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql
438a025d3de0c82a5f58dc96cff8457e3b814f4a98f9732d1cb900c16e144661
[root@localhost home]# docker exec -it mysql01 /bin/bash
root@438a025d3de0:/#  mysql -u root -p123456 

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.01 sec)

# 更改MySQL密码
mysql> ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
Query OK, 0 rows affected (0.00 sec)
# 或者
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456';
```



### Nacos

```shell
docker run --env MODE=standalone --name nacos -d -p 8848:8848 nacos/nacos-server
```



### MongoDB

```shell
docker run --name mymongo -d -p 27017:27017 -v myFolder:/data mongo
```



### Redis

```shell
docker run -itd --name redis-test -p 6379:6379 redis
```



## DockerFile

```shell
FROM ubuntu
RUN apt-get update
RUN apt-get install vim -y
RUN  apt-get install -y net-tools
ENV WORKPATH /usr/local
WORKDIR $WORKPATH
EXPOSE 8888
CMD echo "----end----"
CMD /bin/bash
```



Docker-Compose

```yaml
version: '3'
services:
  kibana:
    expose: 
      - "9200"
    ports: 
      - "5601"
    image: "kibana:7.6.2"
```


