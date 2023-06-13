# Node.js



## 开始

### 安装

```shell
# 安装
# 在存储依赖包的目录下新建文件夹
node_cache
node_global

# 查看global配置目录
# 设置cache和global目录
npm config set prefix "xxx/node_global"
npm root -g
npm config set cache "xxx/node_cache"

# path 配置
NODE_PATH xxx/node_global/node_modules
Path xxx/node_global

# 查看配置
npm config list
# 配置镜像站
npm config set registry=http://registry.npm.taobao.org
# 检查镜像站是否正常
npm config get registry
```

### 使用

```shell
# 查看已安装的依赖包
npm list
npm list -g
npm list --depth=0

# 安装依赖
# 下载到当前目录
# -S 项目运行时使用
# -D 开发中使用
# -g 全局安装
npm install -S -D -g package-name@version

# 卸载依赖
npm uninstall package-name@version
```





## 参考

[Node.js 解压版配置](https://blog.csdn.net/qq_27603235/article/details/79022580)

[设置npm镜像为cnpm](https://www.jianshu.com/p/115594f64b41)
