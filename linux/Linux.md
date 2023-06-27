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
# 查找 redis 开头的文件/目录
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
> ls -l
total 68
lrwxrwxrwx.  1 root root     7 Dec 25 11:18 bin -> usr/bin
dr-xr-xr-x.  5 root root  4096 Feb  7 20:48 boot
drwxr-xr-x  19 root root  2960 Apr 18 10:32 dev
drwxr-xr-x. 79 root root  4096 Apr 18 10:31 etc
drwxr-xr-x.  5 root root  4096 Feb 20 17:03 home

lrwxrwxrwx（共10位）
l -- 表示这是一个链接文件
d -- 表示目录文件
l ->rwx<- rwxrwx 所属主权限
lrwx ->rwx<- rwx 所属组权限
lrwxrwx ->rwx<- 任何人（公有权限）

-rw------- (600)    只有拥有者有读写权限。
-rw-r--r-- (644)    只有拥有者有读写权限；而属组用户和其他用户只有读权限。
-rwx------ (700)    只有拥有者有读、写、执行权限。
-rwxr-xr-x (755)    拥有者有读、写、执行权限；而属组用户和其他用户只有读、执行权限。
-rwx--x--x (711)    拥有者有读、写、执行权限；而属组用户和其他用户只有执行权限。
-rw-rw-rw- (666)    所有用户都有文件读、写权限。
-rwxrwxrwx (777)    所有用户都有读、写、执行权限。
```



### 开发命令

1、查看整机整体性能

```shell
# q 退出top
top
```

- cpu
- memory
- id(Idle) 空闲率，越高越好
- load average 系统负载 1分钟 5 分钟 15 分钟的系统负载量 3 个值相加后除 3 乘 100%  大于 60% 说明系统负担重，大于 80% 说明要完了
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
> vmstat -n 2 3
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
> iostat -xdk 2 3
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



### Docker 命令

```shell
service docker start
service docker stop
service docker restart
```





### 更改 SHELL

```shell
echo $SHELL # 查看当前使用的 shell
cat /etc/shells # 查看已安装的 shell
chsh -s /bin/zsh # 修改 shell 为 zsh
```



**查看 apt 安装位置**

* https://www.cnblogs.com/zhuiluoyu/p/5181098.html

* https://www.cnblogs.com/mch0dm1n/p/5422179.html

* https://blog.csdn.net/u013276277/article/details/81033129



**dpkg 相关命令**

* https://blog.csdn.net/Courage_Insight/article/details/41827167



<br>

## 文件权限

* https://blog.csdn.net/u013197629/article/details/73608613