---
title: docker打包python项目
author: Axellance
date: 2025-04-25
layout: post
category: docker
tags: [docker,软件开发]
mermaid: true
---

使用python进行开发时，会用到很多特定的库，一个成熟的python项目必然会依赖很多特定的环境。然而项目运行的结果不仅取决于代码，和运行代码的环境也息息相关。这很有可能会造成，开发环境上的运行结果和测试环境、线上环境上的结果都不一致的现象。为了解决这个问题，可以将python项目打包成docker镜像，这样即使在不同的机器上运行打包后的项目，我们也能够得到一致的运行结果。因为docker打包是会将项目的代码和环境一起打包。

## 准备python项目

### 1. 项目结构

项目结构如下：

![](../images/docker-arch.png)





## 导出环境依赖

### 1. 导出结果含有路径

将项目中使用的python依赖导出为`requirements.txt`：

```shell
pip freeze > requirements.txt
```

这条命令会将当前环境中所有已安装的包及其版本号导出到`requirements.txt`中，但是这种方式会有本地路径，迁移到别的环境中可能无法使用。这种方式生成的文件如下所示：

```shell
brotlipy==0.7.0
certifi==2020.12.5
cffi @ file:///tmp/build/80754af9/cffi_1606255122322/work
chardet @ file:///tmp/build/80754af9/chardet_1605303192865/work
conda==4.9.2
conda-pack==0.8.1
conda-package-handling @ file:///tmp/build/80754af9/conda-package-handling_1605484832764/work
cryptography @ file:///tmp/build/80754af9/cryptography_1607635322643/work
idna @ file:///tmp/build/80754af9/idna_1593446292537/work
numpy==2.0.2
pandas==2.2.3
protobuf==3.20.3
pycosat==0.6.3
pycparser @ file:///tmp/build/80754af9/pycparser_1594388511720/work
pyOpenSSL @ file:///tmp/build/80754af9/pyopenssl_1606517880428/work
PySocks @ file:///tmp/build/80754af9/pysocks_1605305812635/work
python-dateutil==2.9.0.post0
pytz==2025.2
requests @ file:///tmp/build/80754af9/requests_1606691187061/work
ruamel-yaml-conda @ file:///tmp/build/80754af9/ruamel_yaml_1605527293619/work
six @ file:///tmp/build/80754af9/six_1605205306277/work
tqdm @ file:///tmp/build/80754af9/tqdm_1607369919789/work
tzdata==2025.2
```

### 2. 导出结果不含路径

生成的`requirements.txt`不含本地路径：

```shell
pip list --format=freeze > requirements.txt
```

构建镜像时，将根据`requirments.txt`文件中的内容构建python环境。文件内容如下所示：

```shell
brotlipy==0.7.0
certifi==2020.12.5
cffi==1.14.4
chardet==3.0.4
conda==4.9.2
conda-pack==0.8.1
conda-package-handling==1.7.2
cryptography==3.3.1
idna==2.10
numpy==2.0.2
pandas==2.2.3
pip==20.3.1
protobuf==3.20.3
pycosat==0.6.3
pycparser==2.20
pyOpenSSL==20.0.0
PySocks==1.7.1
python-dateutil==2.9.0.post0
pytz==2025.2
requests==2.25.0
ruamel-yaml-conda==0.15.80
setuptools==51.0.0.post20201207
six==1.15.0
tqdm==4.54.1
tzdata==2025.2
urllib3==1.25.11
wheel==0.36.1
```

我个人一般采用第二种方式。

## 构建docker镜像

其实安装包方式安装docker步骤非常简单，但是安装过程中可能会出现各种各样的错误，所以最好严格按照教程操作，因为安装docker只是第一步，后面通过docker部署各种各样的应用才是我们最主要的目的。

在离线环境下，无法像外网服务器中配置环境那么方便，这个时候使用docker就是最优解。

我们可以在外网中将我们的容器打包：

```shell
docker save -o app.tar <Container ID>
```

然后拷贝到内网加载容器

```shell
docker load -i app.tar
```

如果遇到一个应用包含多个镜像时，每次加载一个镜像就略显僵硬，可以将所有tar包放在一个文件夹下，然后执行以下shell命令：

```shell
for name in `ls ./*.tar`; do docker load -i "$name"; done
```

我的做法是每次加载一个镜像（这是和公司shell大佬学的写法，也算是有所收获），工作中有些时候可以将常用命令写成脚本，这样可以节省一些工作时间，也可以避免一些错误。

## 运行docker容器

我们用docker run运行容器 ，终端显示 hello docker 表示容器运行成功

```shell
sudo docker run 
```

如果想要进行docker的交互式终端查看项目代码，可以通过以下命令：

```shell
sudo docker 
```



## 挂载数据卷

