# Linux



## 命令



### 日常命令

```shell
# 查看系统信息
uname
uname -a

# 安装 neofetch
neofetch

# 安装ps
apt install procps

# 安装systemctl
apt install systemd

# 查找命令
find 目录 -name 查找目标
# 查找redis开头的文件/目录
find 目录 -name "redis*"

# 查看运行情况

# 查看端口号占用情况
lsof -i:端口号

ps -ef|grep 应用名

netstat -anp|grep 端口号

# kill
kill -9 进程id

# 修改权限
chmod +/-X 文件名

# 查看ip情况
ifconfig

# 关闭防火墙
systemctl stop firewalld

# 查看当前目录下文件内容
[root@iZbp141lw91pvis8z44ee5Z /]# ls -l
total 68
lrwxrwxrwx.  1 root root     7 Dec 25 11:18 bin -> usr/bin
dr-xr-xr-x.  5 root root  4096 Feb  7 20:48 boot
drwxr-xr-x  19 root root  2960 Apr 18 10:32 dev
drwxr-xr-x. 79 root root  4096 Apr 18 10:31 etc
drwxr-xr-x.  5 root root  4096 Feb 20 17:03 home
drwxr-xr-x   4 root root  4096 Feb 19 15:41 leyou
lrwxrwxrwx.  1 root root     7 Dec 25 11:18 lib -> usr/lib
lrwxrwxrwx.  1 root root     9 Dec 25 11:18 lib64 -> usr/lib64
drwx------.  2 root root 16384 Dec 25 11:18 lost+found
drwxr-xr-x.  2 root root  4096 Apr 11  2018 media
drwxr-xr-x.  2 root root  4096 Apr 11  2018 mnt
drwxr-xr-x   8 root root  4096 Feb 19 15:25 mydata
drwxr-xr-x.  3 root root  4096 Feb 19 15:53 opt
dr-xr-xr-x  89 root root     0 Apr 18 10:32 proc
dr-xr-x---.  7 root root  4096 Feb 20 17:38 root
drwxr-xr-x  25 root root   780 Apr 18 10:39 run
lrwxrwxrwx.  1 root root     8 Dec 25 11:18 sbin -> usr/sbin
drwxr-xr-x.  2 root root  4096 Apr 11  2018 srv
dr-xr-xr-x  13 root root     0 Apr 18 18:32 sys
drwxrwxrwt.  8 root root  4096 Apr 25 03:35 tmp
drwxr-xr-x. 14 root root  4096 Feb 20 17:21 usr
drwxr-xr-x. 19 root root  4096 Dec 25 03:22 var

lrwxrwxrwx（共10位）
l -- 表示这是一个链接文件
d -- 表示目录文件
l ->rwx<- rwxrwx 所属主权限
lrwx ->rwx<- rwx 所属组权限
lrwxrwx ->rwx<- 任何人（公有权限）
```



### 开发常用命令

1、查看整机整体性能

```shell
# q退出top
top
# top精简版

```

- cpu
- memory
- id(Idle) 空闲率，越高越好
- load average 系统负载 1分钟 5分钟15分钟的系统负载量 3个值相加后除3乘100% 大于60%说明系统负担重，大于80%说明要完了
- uptime

2、查看内存

```shell
# （完整版，显示字节）
free

# （四舍五入版）
free -g

# （常用）
free -m
```

3、查看硬盘

```shell
# （disk free，显示字节）磁盘使用情况
df

# （disk free -human，常用）
df -h
```

4、CPU

```shell
#查看cpu使用情况（包含但不限于cpu，还包括进程，内存），每隔2秒刷新，共显示3行记录
[root@iZbp141lw91pvis8z44ee5Z ~]# vmstat -n 2 3
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 6  0      0 1018380 136920 492228    0    0     1     2   23   39  1  1 97  0  0
 0  0      0 1018380 136920 492268    0    0     0     0  319  900  1  1 99  0  0
 0  0      0 1017968 136920 492268    0    0     0     0  296  893  1  1 98  0  0
 procs
 r -- running 正在运行的进程
 b -- blocked 堵塞进程
 cpu
 us -- user
 sy -- system
 id -- idle（空闲率，越高越好）
```

5、IO

```shell
#每隔2秒刷新，共显示3行记录
[root@iZbp141lw91pvis8z44ee5Z ~]# iostat -xdk 2 3
Linux 3.10.0-1062.12.1.el7.x86_64 (iZbp141lw91pvis8z44ee5Z)     04/25/2020      _x86_64_        (1 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.00     0.09    0.03    0.29     0.76     1.56    14.81     0.00    3.24   10.10    2.58   0.44   0.01

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.00     1.52    0.51    1.01     4.04    10.10    18.67     0.02   11.00   31.00    1.00  10.67   1.62

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

r/s	-- 每秒磁盘读   
w/s -- 每秒磁盘写
%util -- （越小越好）
```



#### docker

```shell
service docker start
service docker stop
service docker restart
```





### 系统命令

```shell
echo $SHELL # 查看当前使用的shell
cat /etc/shells # 查看已安装的shell
chsh -s /bin/zsh # 修改shell为zsh，修改完成需要重启终端，最后安装oh-my-zsh
```



**查看apt-get install 安装位置**

https://www.cnblogs.com/zhuiluoyu/p/5181098.html

https://www.cnblogs.com/mch0dm1n/p/5422179.html

https://blog.csdn.net/u013276277/article/details/81033129

**dpkg 相关命令**

https://blog.csdn.net/Courage_Insight/article/details/41827167