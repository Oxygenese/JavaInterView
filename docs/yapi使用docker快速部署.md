在我们实际项目开发中，需要对越来越多的api进行管理，而且我们往往是前后端分离开发，甚至更加大型的项目需要协作开发，因此对于统一api的管理十分重要。
今天给大家介绍一款十分好用的api管理系统——yapi！
如果直接在主机上部署，不仅麻烦，而且管理十分不便，我们这里采用docker容器化部署！

## 一、[yapi官网wiki教程](https://hellosean1025.github.io/yapi/devops/index.html)

## 二、docker容器化部署

### 1.安装docker以及docker-compose

```shell
自行百度或见往期教程
```

### 2.将仓库克隆到本地

```shell
git clone git@gitee.com:zhou_gong_hao/docker-YApi.git
cd docker-YApi
```

### 3.修改初始配置

主要修改   YAPI_ADMIN_ACCOUNT和 YAPI_ADMIN_PASSWORD这两个属性。

```shell
#编辑docker-compose.yml文件
vim docker-compose.yml
```

```shell
#保存并退出
:wq
```

```shell
#执行
docker-compose up -d
#等待镜像抓取并运行成功
```

### 4.部署成功并访问

```shell
ip:40001
```



