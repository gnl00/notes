# Python



## 0 安装

1、安装并配置好 Python 环境

2、安装 MiniConda/AnaConda

1）下载：`https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/`

2）配置 conda 源 `~/.condarc`

```yaml
show_channel_urls: true
channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/menpo/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/bioconda/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/msys2/
  - http://mirror.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
  - http://mirror.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
  - defaults
```

3）进行 conda 操作

```shell
# 查看环境
conda env list

# 创建环境
conda create -n test python=3.7

# 切换环境，切换到其他环境必须是在 base 环境下
source activate test

# 安装包
conda install xlrd

# 退出环境
conda deactivate test

# 删除环境
conda remove -n test --all
```

3、安装 pytorch：`https://pytorch.org/`

```shell
# 记得去掉 -c pytorch，否则不走自定义镜像地址
conda install pytorch torchvision torchaudio cudatoolkit=11.3
```

4、测试是否支持使用 GPU 进行模型训练（Nvidia GPU软件版本需要高于392）

```python
import torch
torch.cuda.is_available()

// 返回True说明支持
```

5、conda 安装 JupyterNoteBook

```shell
conda install nb_conda
```

6、常用命令

```python
# dir() 输出 torch.cuda 下包含的内容
dir(torch.cuda)
# 
help(torch.cuda.is_available)
```





## 1 预备知识

### 1.1 线性代数基础

### 1.2 微分基础

### 1.3 概率论基础





## 2 线性神经网络























































































**参考**

[miniconda安装及使用 - -至尊宝- - 博客园 (cnblogs.com)](https://www.cnblogs.com/zhizunbao-monky/p/14052479.html)

[conda的安装与使用（2021-04-27更新） - 简书 (jianshu.com)](https://www.jianshu.com/p/edaa744ea47d)