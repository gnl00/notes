# Ubuntu安装



install ubuntu (safe graphics)



### 分区

| 挂载点                              | 分区类型        | 文件系统 | 大小               | 空间起始位置        |
| ----------------------------------- | --------------- | -------- | ------------------ | ------------------- |
| null                                | 逻辑分区logical | efi      | 500M               | 空间起始点beginning |
| null                                | 主分区primary   | swap     | 与机器物理运存一致 | 空间起始点beginning |
| /boot（UEF可不设boot，le'ge'a'c'y） | 逻辑分区logical | ext4     | 200M               | 空间起始点beginning |
| /home                               | 逻辑分区logical | ext4     | 10GB               | 空间起始点beginning |
| /                                   | 逻辑分区logical | ext4     | 剩余空间           | 空间起始点beginning |



### 突然卡屏问题

重启系统，在GUN GRUB界面，选择 `advanced options for Ubuntu` 进入，

选择 `recover mode` 进入，

通过方向键上下回车选择 `root Drop to root shell prompt` 

```shell
#注：需先解决vi编辑器的问题
vi /etc/vim/vimrc.tiny

:set nocompatible
:set backspace=2
按a或i进入编辑模式
set nocompatible （原为set compatible）
set backspace=2
:wq (保存并退出)

#####################################################
#输入命令
#以读写权限重新挂载系统
mount -o remount /
#编辑grub
vi /etc/default/grub
#将文件中的
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
#修改为
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nomodeset"
#保存退出
#再输入
update-grub
#更新grub信息
#按 Ctrl+D 回到界面
#选择 resume Resume normal boot 出现新界面（OK） 回车进入系统
```



安装： https://blog.csdn.net/a845717607/article/details/98449419

deepin DRM:Poiter to TMDS table invilid https://blog.csdn.net/yaorengjin0808/article/details/81613331

1050ti nvidia https://blog.csdn.net/u014003662/article/details/88547512

其他 nVidia https://blog.csdn.net/chentianting/article/details/85089403

网速 https://blog.csdn.net/u012995500/article/details/104426042

删除bios 其他启动项 https://blog.csdn.net/qq_35379989/article/details/85941975



### 显示字体

https://blog.csdn.net/qq_16056397/article/details/105732802

https://blog.csdn.net/baidu_41560343/article/details/87796946



### 配置显卡



ctrl+F4 进入tty编辑器模式



https://blog.csdn.net/qingdu007/article/details/97306651

https://blog.csdn.net/u014003662/article/details/88547512

https://blog.csdn.net/weixin_38423311/article/details/80965594



若是安装完显卡之后一直循环锁屏界面无法进入



1、删除显卡配置

https://blog.csdn.net/weixin_43981221/article/details/90113079

2、将 /etc/default/grub 

找到GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"

修改回原来的状态





### 配置软件源

阿里 

https://blog.csdn.net/wangyijieonline/article/details/105360138

https://blog.csdn.net/YooLcx/article/details/104527734



### 配置 JDK



https://blog.csdn.net/xiamoyanyulrq/article/details/83022632

https://blog.csdn.net/zbj18314469395/article/details/86064849

在线

https://blog.csdn.net/qq_41813208/article/details/100980368

离线

https://blog.csdn.net/ljp345775/article/details/104595904

https://blog.csdn.net/weixx3/article/details/94109337



apt-get install openjdk-8-java 安装的目录为 /usr/local/lib/Java

解压包自定义目录一般解压到opt目录



编辑环境变量

```shell
# 方式1
vim /etc/profile
# 方式2
~/.bashrc
```



```shell
# 配置jdk1.8环境
export JAVA_HOME=/usr/local/lib/java/jdk1.8.0_291
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH

# JDK11
export JAVA_HOME=/usr/local/lib/jdk-11.0.11
export PATH=${JAVA_HOME}/bin:$PATH
```

使环境变量生效

```shell
source /etc/profile
```



### 配置maven

```shell
export MAVEN_HOME=/opt/maven/apache-maven-3.6.3
export CLASSPATH=$CLASSPATH:$MAVEN_HOME/lib
export PATH=$PATH:$MAVEN_HOME/bin
```

刷新配置

```shell
source /etc/profile
```



### 配置 Python





### 配置 Nodejs

https://www.cnblogs.com/feiquan/p/11223487.html



### 配置 Git

https://www.cnblogs.com/sunshinekevin/p/11617562.html



### 配置 Maven

https://blog.csdn.net/qq_29695701/article/details/90705181



### 配置docker

https://www.jianshu.com/p/80e3fd18a17e