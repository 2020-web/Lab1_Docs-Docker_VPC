# 高级Web技术Lab 1： 申请服务器+创建VPC+Docker部署

## 概述

 Lab1的主要内容包括：

* 在AWS中申请自己的服务器
* 创建带有公网和私有子网的VPC
* **重点掌握Docker**，并使用Docker在服务器上部署一个简易的JAVA程序
## Part 1：申请服务器

云服务器(Elastic Compute Service, ECS)是一种简单高效、安全可靠、处理能力可弹性伸缩的计算服务。其管理方式比物理服务器更简单高效。用户无需提前购买硬件，即可迅速创建或释放任意多台云服务器。

AWS对于信用卡使用者提供一定的免费时长，如果是教育用户则采取事先给予一定使用额度的方法，同学们在使用的时候要注意以下几点：

1、实例模板ec2.micro的创建是免费的，但是一个月的总使用时长只有750小时，这对于一台实例是完全够用一个月的，但是对于两台及两台以上就不够了。

2、申请弹性IP以后，如果不适用，也是会扣额度的。

具体的服务器申请指导文档见**./assets/material/Amazon AWS 使用教程.pdf**。文档中的注册地址可能已经过时，[见此](https://www.awseducate.com/registration#APP_TYPE)。

## Part 2： 创建带有公网和私有子网的VPC

虚拟私有云（Virtual Private Cloud, VPC）是存在于共享或公用云中的私有云，亦即一种网络云。借助VPC，可以在AWS云中预置一个逻辑隔离的部分，从而在自己定义的虚拟网络中启动AWS资源。你可以完全掌控这个虚拟联网环境，包括选择自己的IP地址范围、创建子网以及配置路由表和网络网关。

在[官方文档](https://docs.aws.amazon.com/zh_cn/vpc/latest/userguide/VPC_Scenario2.html)中有推荐的vpc配置，这里摘录重要的两段：

> 这个场景的配置包括一个有公有子网和私有子网的 Virtual Private Cloud (VPC)。如果您希望运行面向公众的 Web 应用程序，并同时保留不可公开访问的后端服务器，我们建议您使用此场景。常用例子是一个多层网站，其 Web 服务器位于公有子网之内，数据库服务器则位于私有子网之内。您可以设置安全性和路由，以使 Web 服务器能够与数据库服务器建立通信。

> 公有子网中的实例可直接将出站流量发往 Internet，而私有子网中的实例不能这样做。但是，私有子网中的实例可使用位于公有子网中的网络地址转换 (NAT) 网关访问 Internet。数据库服务器可以使用 NAT 网关连接到 Internet 进行软件更新，但 Internet 不能建立到数据库服务器的连接。

同学们可以按照这个默认的配置图配置。

![带有公有子网和私有子网的 VPC 			](https://docs.aws.amazon.com/zh_cn/vpc/latest/userguide/images/nat-gateway-diagram.png)





关于“AWS VPC with Public and Private Subnets”的关键词在Youtube上有大量的视频演示配置操作，推荐同学们看[这个](https://www.youtube.com/watch?v=wG9En2X9FvM)。

**这里为同学们摘录关键步骤：**

1、使用VPC Wizard创建VPC。在这个界面，需要切换使用NAT实例（使用NAT网关会报错“没有权限”），然后为“VPC名称”，“公有子网名称”和“私有子网名称”起一个自己可以标识的名字。其他的都可以按默认配置。

![image-20200310222918814](assets/image/image-20200310222918814.png)

创建完之后，切换到服务EC2，可以看到已经有台ec2实例，这台实例就是NAT实例。

2、为NAT实例创建安全组

NAT实例默认使用“default”安全组，并不符合要求，我们要自己创建并配置一个安全组。

在服务VPC中，选中“安全组”选项卡，之后创建安全组。

![image-20200311161010560](assets/image/image-20200311161010560.png)

填上关键信息，并在VPC选项中选中之前创建好的VPC。

![image-20200311161207474](assets/image/image-20200311161207474.png)

创建完之后，选中安全组，创建如下规则。安全组可以控制流量出入，在本文中，为了方便和日后开发，统一采用粗粒度的设置方式，如果有同学想要更细粒度的管理，请自行配置。

**Inbound Rules**

| 类型     | 协议 | 端口 | 来源        |
| -------- | ---- | ---- | ----------- |
| 所有流量 | 全部 | 全部 | 10.0.1.0/24 |
| SSH      | TCP  | 22   | 0.0.0.0/0   |

**Outbound Rules**

| 类型     | 协议 | 端口 | 目的地    |
| -------- | ---- | ---- | --------- |
| 所有流量 | 全部 | 全部 | 0.0.0.0/0 |

10.0.1.0/24是内网网段，如果有同学采用非默认配置，注意更改。

我们在这里设置所有从内网网段中的流量都经过NAT实例转发，之后再将安全组NATSG与NAT实例绑定。

3、分别再启动两个实例，系统选择ubuntu18，分别加入到公网和私网中，公网中的实例要勾选自动分配IP，如果在这里漏掉，可以在创建完实例之后为其分配弹性IP。

![image-20200310224132969](assets/image/image-20200310224132969.png)

可以对机器打tag，避免混淆

![image-20200310223508016](assets/image/image-20200310223508016.png)

在创建的最后需要创建新的安全组并设置规则。这里给出推荐配置。

`公网实例安全组WebServerSG`

**Inbound Rules**

| 类型 | 协议 | 端口 | 来源      |
| ---- | ---- | ---- | --------- |
| HTTP | TCP  | 80   | 0.0.0.0/0 |
| SSH  | TCP  | 22   | 0.0.0.0/0 |

Outbound Rules

| 类型     | 协议 | 端口 | 目的地    |
| -------- | ---- | ---- | --------- |
| 所有流量 | 全部 | 全部 | 0.0.0.0/0 |

`私网实例安全组DBServerSG`

**Inbound Rules**

| 类型     | 协议 | 端口 | 来源        |
| -------- | ---- | ---- | ----------- |
| 所有流量 | 全部 | 全部 | 10.0.0.0/24 |

**Outbound Rules**

| 类型     | 协议 | 端口 | 目的地    |
| -------- | ---- | ---- | --------- |
| 所有流量 | 全部 | 全部 | 0.0.0.0/0 |

4、与内网实例连接

与公网实例的连接在PART1中“申请服务器”的实验中，同学们已经体验过了，现在要与内网实例连接。

选择通过公网实例连接到内网实例，如果有同学想要直接连接到内网实例，请自行尝试。

把密钥文件传输到公网实例中，先设置密钥权限，然后通过ssh命令连接到内网实例。

```shell
chmod 400 aws_lab.pem

ssh ubuntu@10.0.1.197 -i ./aws_lab.pem
```

10.0.1.197是内网实例的私有IP。

连接到内网实例后，如果这个时候还不能与因特网连接，那就需要配置路由表

5、配置路由表

![image-20200311151103225](assets/image/image-20200311151103225.png)

在“服务VPC”——路由表选型卡中，选中之前创建的VPC的主路由表，点击“编辑子网关联”，在选项页面中选中私有子网，将私有子网与VPC主路由表相连。

此时内网实例应该能与因特网连通，但是无法被外网访问到，只能通过VPC中的公有网段内的实例访问。

![image-20200311160232305](assets/image/image-20200311160232305.png)

至此，我们完成了VPC的配置。

## Part 3：使用Docker

实验要求：配置Docker并成功部署所给的简易java项目，项目源代码[见此](https://github.com/2020-web/lab1_Code-docker_demo)。

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的镜像中，然后发布到任何流行的 Linux或Windows 机器上，也可以实现虚拟化。

推荐同学们先去阅读**./assets/material/**文档中提供的三个文档，对Docker建立大概的了解。

### 1.准备Docker环境

下面演示在服务器环境是 Ubuntu 18.04 下安装Docker，既然采用的是亚马逊云服务器，服务器在国外，可以使用Docker官方文档的步骤来安装Docker。

链接[见此](https://docs.docker.com/install/linux/docker-ce/ubuntu/)。

**注意：**安装的是Docker CE版本。

### 2.编写Dockerfile

```shell
mkdir base-tomcat-maven

cd base-tomcat-maven

vim Dockerfile
```

```Dockerfile
FROM maven:3.3.3
ENV CATALINA_HOME /usr/local/tomcat
ENV PATH $CATALINA_HOME/bin:$PATH
RUN mkdir -p "$CATALINA_HOME"
WORKDIR $CATALINA_HOME
ENV TOMCAT_VERSION 8.5.51
ENV TOMCAT_TGZ_URL https://www.apache.org/dist/tomcat/tomcat-8/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz
RUN set -x \
&& curl -fSL "$TOMCAT_TGZ_URL" -o tomcat.tar.gz \
&& tar -xvf tomcat.tar.gz --strip-components=1 \
&& rm bin/*.bat \
&& rm tomcat.tar.gz*
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

解释：在一个maven基础镜像上叠加tomcat，最终运行java项目时只需要这一个镜像即可完成编译+打包+部署。

### 3.build镜像

在上一步的目录中执行：

```shell
sudo docker build -t base-tomcat-maven .
```

来构建镜像。使用`-t`参数指定镜像名称:标签，`.`表示使用当前目录下的 Dockerfile, 还可以通过-f 指定 Dockerfile 所在路径。

此时运行`docker images`应该可以看到一个叫`base-tomcat-maven`的镜像。

基于此镜像， 将java应用通过maven编译打包到tomcat webapps目录，生成最终镜像。

```shell
git clone https://github.com/2020-web/lab1_Code-docker_demo.git

cd lab1_Code-docker_demo

sudo docker build -t docker-demo .
```

### 4.运行容器

基于镜像，运行容器

```shell
# 创建并启动一个名为demo的docker容器，主机的8001端口映射docker容器的8080端口
sudo docker run -idt --name demo -p 8001:8080 docker-demo:latest
# 模拟HTTP请求，测试浏览器是否能正常访问  
curl localhost:8001
```

### 5.安全组问题

如果使用 AWS 服务器，默认的安全组规则会拦截服务器的入站流量。为了能够正常访问，我们需要放开对 8001 端口的限制：

1. 进入 AWS 控制面板，点击左边的【安全组】
2. 点击下方的【入站】->【编辑】
3. 增加入站规则： HTTP 或 HTTPS 可以允许他人访问你服务器上的网址；自定义 TCP 端口 8001 ，允许来源 【任意位置】。

如果一切正常，就可以通过 服务器公网 IP:8001 浏览器来访问你的项目了。

## Part 4：提交

截止时间：2020/3/31-23:59:59

提交方式：将文档提交到超星指定的lab作业栏里，文档格式不限。





