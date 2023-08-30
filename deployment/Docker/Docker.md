# Docker



## 安装 Docker

> Windows 先安装 WSL2，再安装 Docker。

> 镜像加速与修改镜像源：
>
> * [阿里云服务器设置 Docker 镜像加速器](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)
> * [更改docker仓库源地址 1](https://blog.csdn.net/m0_37886429/article/details/80323149)
> * [更改docker仓库源地址 2](https://www.jianshu.com/p/df75f9b5fcf6)



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



### 非 root 用户启动

* [manager docker as a non-root user](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user)



## 镜像

### MySQL

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



### Maven 私服

```shell
docker run -d --name nexus -p 8081:8081  -v /srv/nexus-data:/var/nexus-data --restart=always sonatype/nexus3:latest
```



### Alpine

> apline 镜像换源：https://www.jianshu.com/p/791c91b7c2a4



## Dockerfile

> https://docs.docker.com/develop/develop-images/dockerfile_best-practices/

### Ubuntu

```dockerfile
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



### Java 应用

```dockerfile
FROM openjdk:8-jre-alpine

# 更换 apline 镜像源
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories

# 设置镜像内时区
RUN apk add tzdata
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo "Asia/Shanghai" > /etc/timezone
RUN apk del tzdata

RUN apk add ffmpeg
RUN apk add vim

WORKDIR application
# jar 相对 Dockerfile 所在的位置
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} application.jar

# Forward logs to Docker
# RUN ln -sf /dev/stdout /application/logs/media-sfu.log && \
#    ln -sf /dev/stderr /application/logs/media-sfu-error.log

ENTRYPOINT ["java", "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=0.0.0.0:5005", "-jar", "application.jar"]
```



### nginx-rtmp

```dockerfile
FROM alpine:latest

ARG BUILD_DIR=/tmp/build

ARG NGINX_CONF=nginx.conf
ARG NGINX_CONF_DIR=/etc/nginx

ARG NGINX_VERS=nginx-1.25.2
ARG NGINX_RTMP_MODULE_VERS=nginx-rtmp-module-1.2.2
ARG ZLIB_VERS=zlib-1.3
ARG PCRE_VERS=pcre2-10.42
ARG OPENSSL_VERS=openssl-3.0.10

ARG NGINX_FILE=${NGINX_VERS}.tar.gz
ARG NGINX_RTMP_MODULE_FILE=${NGINX_RTMP_MODULE_VERS}.tar.gz
ARG ZLIB_FILE=${ZLIB_VERS}.tar.gz
ARG PCRE_FILE=${PCRE_VERS}.tar.gz
ARG OPENSSL_FILE=${OPENSSL_VERS}.tar.gz

# 换源
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories

#RUN  sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list && \
#     sed -i s@/security.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list && \
#     apt clean && \
#     apt update -y

RUN mkdir -p ${BUILD_DIR}

# download and decompress nginx
# RUN apt install -y -y wget
# RUN wget -O nginx.tar.gz https://nginx.org/download/nginx-1.25.2.tar.gz &&

# download and decompress nginx-rtmp-module
# RUN wget -O nginx-rtmp-module.tar.gz https://github.com/arut/nginx-rtmp-module/archive/refs/tags/v${NGINX_RTMP_MODULE_VERSION}.tar.gz && tar -zxf nginx-rtmp-module.tar.gz

# download extract package
# RUN wget -O ${ZLIB_VERS}.tar.gz http://zlib.net/${ZLIB_VERS}.tar.gz && tar -zxf ${ZLIB_VERS}.tar.gz

COPY ${NGINX_FILE} ${BUILD_DIR}/
COPY ${NGINX_RTMP_MODULE_FILE} ${BUILD_DIR}/
COPY ${ZLIB_FILE} ${BUILD_DIR}/
COPY ${PCRE_FILE} ${BUILD_DIR}/
COPY ${OPENSSL_FILE} ${BUILD_DIR}/
COPY ${NGINX_CONF} ${BUILD_DIR}/

RUN cd ${BUILD_DIR} && \
    tar -xvf ${NGINX_FILE} && \
    tar -xvf ${NGINX_RTMP_MODULE_FILE} && \
    tar -xvf ${ZLIB_FILE} && \
    tar -xvf ${PCRE_FILE} && \
    tar -xvf ${OPENSSL_FILE} && \
    rm -rf *.tar.gz

RUN cd ${BUILD_DIR} && rm -rf *.tar.gz

# install compile tools and dependencies, compile nginx with nginx-rtmp-module
RUN apk add --no-cache gcc g++ make perl linux-headers && \
    cd ${BUILD_DIR} && cd ${NGINX_VERS} && \
    ./configure \
    --sbin-path=/usr/local/sbin/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --http-log-path=/var/log/nginx/access.log \
    --error-log-path=/var/log/nginx/error.log \
    --pid-path=/var/run/nginx/nginx.pid \
    --lock-path=/var/lock/nginx/nginx.lock \
    --http-client-body-temp-path=/tmp/nginx-client-body \
    --with-http_ssl_module \
    --with-zlib=../${ZLIB_VERS} \
    --with-pcre=../${PCRE_VERS} \
    --with-openssl=../${OPENSSL_VERS} \
    --add-module=../${NGINX_RTMP_MODULE_VERS} && \
    make && \
    make install && \
    # cp custom conf
    cp ../${NGINX_CONF} ${NGINX_CONF_DIR} && \
    # forward logs to docker
    ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log && \
    apk del gcc g++ make linux-headers && apk add --no-cache vim && \
    rm -rf ./* && \
    rm -rf ${BUILD_DIR} &&\
    rm -rf /var/cache/apk/*

EXPOSE 80
EXPOSE 90
EXPOSE 8080
EXPOSE 9090
EXPOSE 1935

CMD ["nginx", "-g", "daemon off;"]
```



## Docker Compose

### kibana

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



## Docker 镜像瘦身

> * https://docs.erda.cloud/blog/post/2021/07/15/docker-compression/ dive 工具的使用
> * https://juejin.cn/post/7074981052233711647 基础镜像的选择
> * https://zhuanlan.zhihu.com/p/161685245 Run 指令合并



## Docker 私有仓库

```shell
docker run -d --name registry -p 5000:5000 -v /srv/docker-registry:/var/lib/registry --restart=always registry
```



**配置非 https 仓库地址**

编辑 `/etc/docker/daemon.json`

```json
{
  "insecure-registries": [
    "<your-registry-host>:<your-registry-port>"
  ]
}
```



**镜像发布**

```shell
docker tag ubuntu:latest 127.0.0.1:5000/ubuntu:latest
docker push 127.0.0.1:5000/ubuntu:latest
```



**查看仓库中的镜像**

```shell
> curl 127.0.0.1:5000/v2/_catalog
{"repositories":[""]}
```
