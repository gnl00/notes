# Maven

JAR 包安装到本地仓库

```shell
# 将本地 jar 包安装到本地依赖库
mvn install:install-file -DgroupId=org.springframework.cloud -DartifactId=spring-cloud-starter-bus-amqp -Dversion=2.2.1.RELEASE -Dpackaging=jar -Dfile=fastdfs-client-1.27.2.jar

# -DgroupId=com.alibaba	--groupId
# -DartifactId=druid-spring-boot-starter	--artifactId
# -Dversion=1.1.21	--version
# -Dfile=druid-spring-boot-starter-1.1.21.jar	--要安装的jar包名
```

