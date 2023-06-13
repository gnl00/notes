# Manjora安装



https://www.jianshu.com/p/f27c70245985

https://blog.csdn.net/lj402159806/article/details/80218360

https://www.jianshu.com/p/333cf36d0914



## 分区

将 boot/efi 分区挂载到win系统格式为 fat32的100M的efi分区中 

交换分区

home目录

/目录



## 驱动

开机卡死每次都会遇到

```
sudo vi /ect/default/grub 
```

在 quiet 后加上 nouveau.modeset=0，然后

```
 
 sudo update-grub
```

先换源，后在设置中安装好驱动





##                  [manjaro 设置 国内源](https://www.cnblogs.com/lemos/p/7640680.html)             

注意，如果安装过程中出现无法连接服务，请参看第 4条。

\1. 设置官方镜像源（包括 core， extra， community， multilib ）

```
$ sudo pacman-mirrors -i -c China -m rank //更新镜像排名
$ sudo pacman -Syy //更新数据源
```

更新 archlinux 签名

pacman -S archlinux-keyring 

 

2.设置 archlinuxcn 源

修改 /etc/pacman.conf　　=> 末尾添加

```
[archlinuxcn]
SigLevel = Optional TrustedOnly
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

 

安装 PGP 签名

1>屏蔽签名

修改/etc/pacman.conf，将原有的SigLevel=××××××注释掉，添加SigLevel = Never即可。

2>重新签名（这步先跳过，直接执行第3步）

```
pacman -Syu haveged
systemctl start haveged
systemctl enable haveged
rm -rf /etc/pacman.d/gnupg
pacman-key --init
pacman-key --populate manjaro
pacman-key --populate archlinuxcn
```

3>安装 archlinuxcn源（或相应源）的签名

```
$ sudo pacman -Syy
$ sudo pacman -S archlinuxcn-keyring 
```

4>运行 以下命令，更新数据源索引及源里的包（此时可还原之前的 SigLevel ）

```
$ sudo pacman -Syy 
$ yaourt -Syua
```

 

3.设置 aur 源

修改 /etc/yaourtrc，去掉 # AURURL 的注释，修改为

```
AURURL="https://aur.tuna.tsinghua.edu.cn"
```

 

4.注意执行以上操作时，可能需要关闭系统的网络代理（如果之前设置过网络代理的话）。



建议安装一个aur helper，例如yay，记得同时安装base-devel，内含一些工具

sudo pacman -Ss yay base-devel

    1 sudo pacman -Ss yay base-devel

然后配置AUR的源，可开启色彩输出

```bash
yay --aururl "https://aur.tuna.tsinghua.edu.cn" --save
sudo sed -i "s/#Color/Color/g" /etc/pacman.conf
```

yay的常见命令和pacman类似



https://www.jianshu.com/p/966017a6f251







## Manjaro/Arch添加archlinuxcn源

编辑 `sudo vim /etc/pacman.conf` 并添加以下内容

```csharp
[archlinuxcn]
# The Chinese Arch Linux communities packages.
SigLevel = Optional TrustAll
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

```
sudo pacman -Syu
```



添加源以及安装输入法 https://www.jianshu.com/p/d64c4d84abe5



## 安装输入法



搜狗不显示候选窗口 https://blog.csdn.net/a924068818/article/details/105215300/



## 密匙操作

1.首先更新一下密钥，如果没有安装archlinux-keyring,请及时安装

```
sudo pacman-key --refresh-keys
```

2.重新加载相应的签名密钥

```
sudo pacman-key --init
sudo pacman-key --populate
```

3。清除pacman 的缓冲文件

```
sudo pacman -Scc
```

4.更新或者安装系统即可

```
sudo pacman -Syu
```



## 安装常用软件

安装chrome之前需要先配置archlinuxcn源

[Manjaro 安装 jdk 8 / docker / docker compose 等常用软件记录](https://hacpai.com/article/1561016470769)



https://www.cnblogs.com/liumanghanfei/p/11153414.html



https://blog.tshine.me/manjaro-install-and-setup/



https://blog.csdn.net/zbgjhy88/article/details/85110220



# manjaro 更新报错-无效或已损坏的软件包 (PGP 签名)

https://blog.csdn.net/weixin_43968923/article/details/86517381



## Pacman常用命令

https://www.jianshu.com/p/ea651cdc5530