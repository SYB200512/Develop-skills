# Docker

## 一、核心概念

Docker是一种软件部署技术，利用容器化技术为应用程序封装独立的运行环境。每个运行环境即为一个**容器**，承载容器运行的计算机称为**宿主机**。

容器与虚拟机的区别：

1.Docker容器: 多个容器共享同一个系统内核。

2.虚拟机: 每个虚拟机包含一个操作系统的完整内核。

3.优势: Docker容器比虚拟机更轻量、占用空间更小、启动速度更快。

镜像 (Image)是容器的模板，可类比为软件安装包。镜像类似于制作糕点的模具，可用于创建多个糕点（容器），并可分享给他人。

- 容器是基于镜像运行的应用程序实例，可类比为安装好的软件。

**Docker仓库** (Registry): 用于存放和分享Docker镜像的场所。

-  Docker Hub: Docker的官方公共仓库，存储了大量用户分享的Docker镜像。

**二、Docker安装**

Docker是基于Linux的容器化技术。在Windows和Mac电脑上，Docker通过虚拟化一个Linux子系统来运行。

Linux系统宿主机是最佳的Docker实战环境。

- **Linux系统安装**

1. 访问`getdocker.com`获取安装脚本。

2. 执行安装脚本（例如，通过`curl -fsSL https://get.docker.com -o get-docker.sh`下载脚本，然后执行`sudo sh get-docker.sh`）。

 \3. 安装完成后，若非`root`用户，需在所有`docker`命令前添加`sudo`以获取管理员权限。

**三、docker pull 下载镜像命令**

用于从Docker仓库下载镜像到本地。

一个完整的镜像名称包含四部分内容：**registry/namespace/image_name:tag**

- **registry**: 仓库地址。`docker.io`表示Docker Hub官方仓库，官方仓库可省略。
- **namespace**: 命名空间，通常是作者或组织名称。`library`是Docker官方仓库的命名空间，可省略。
- **image_name**: 镜像的名称。
- **tag**: 镜像的标签名，通常表示版本号。`latest`表示最新版本，可省略。

示例 docker pull nginx：从Docker Hub官方仓库下载最新版Nginx镜像。

docker pull docker.n8n.io/n8nio/n8n: 从n8n的私有仓库下载n8n镜像。

**Docker Hub网站：Docker官方仓库**，可搜索、查看镜像详情（如官方镜像、版本号、使用说明）。

**Registry (仓库地址/注册表) 与** **Repository** **(镜像库)**：

- repository=registry/namespace/image_name
- 一个镜像库中存放同一镜像的不同版本
- 整个Docker Hub网站可视为一个Registry。
- Nginx可视为一个Repository

 下载镜像时因网络问题报错（Linux/Windows/Mac）：06:57

**docker images：**列出所有已下载到本地的Docker镜像。

**docker rmi image_name 或 docker rmi image_id**：删除镜像

- 可指定镜像名称或ID。

**docker pull** **--platform=xxx** **nginx：**拉取特定CPU架构的镜像

- Docker镜像作为软件，在不同的CPU架构下（如AMD64、ARM64）有不同的版本。
- docker pull`命令默认会自动选择最适合当前宿主机CPU架构的镜像，因此大部分情况下不需要关注这个选项。
- 特殊情况：对于某些低功耗迷你主机（如香橙派），其CPU架构通常为ARM64，需提前确认所需镜像是否提供ARM64版本。



**四、docker run运行容器**

**docker run 镜像名或镜像id：使用镜像创建并运行容器 (最重要命令)**

- 用模具制造一个糕点/进行类的实例化

**docker ps: 查看正在运行的容器。**docker run后，原窗口被占用，可新开窗口执行docker ps

- ps：process status（进程状态）
- 输出信息包括：Container ID（容器唯一ID）、Image（基于哪个镜像创建）、Names（容器名称，如果docker run没有手动设置名字，则系统会自动分配一个随机的名字）。
- docker ps默认只查看运行中的容器
- docker ps **-a**：支持查看运行和已停止的容器

**docker run** **-d** **镜像名或镜像id**

- d：detached mode，分离模式，表示让容器在后台运行，不会阻塞当前窗口。控制台只打印容器ID，容器日志不会直接输出到终端。
- 执行后打印的是容器的长ID，而docker ps会输出容器的一个短ID，短ID就是截取的长ID，作用是一样的

**自动拉取：**可以跳过docker pull命令，直接执行docker run，如果本地不存在指定镜像，docker run会先自动拉取镜像，再创建并运行容器。

**docker run** **-p** **端口号1:端口号2 镜像名**

- 每个容器运行在独立的虚拟网络环境中，与宿主机的网络隔离，默认无法直接从宿主机访问容器内部网络。
- -p参数将宿主机的端口映射到容器内部的端口。
- -p 宿主机端口:容器内部端口（先外后内）

  \* **示例**: `-p 80:80`将宿主机的80端口转发到容器内的80端口。



**五、挂载卷**

**【绑定挂载】docker run** **-v** **宿主机目录:容器内目录 镜像名**

- 功能: 将宿主机的文件目录与容器内的文件目录进行绑定。使得在任一方修改该文件夹时，另一方都会同步修改
- 挂载卷：绑定的目录就称为挂载卷
- 目的: 实现数据的**持久化保存**。当容器被删除时，容器内的数据也会被删除，但**挂载卷可确保容器删除时，数据仍保存在宿主机上**。

**【命名卷挂载】****docker run** **-v** **卷的名字:容器内目录 镜像名**

- 让Docker自动创建一个存储空间，并为其命名：docker volume create 卷的名字
- 查询挂载卷在宿主机的真实目录：docker volume inspect 卷的名字
- 命名卷挂载的好处：命名卷第一次使用的时候，docker会把容器目录中的内容同步到命名卷中，进行一个初始化；而绑定挂载没有这个功能

**docker volume list：**查看所有创建过的卷

**docker volume rm 卷名：**删除卷

**docker volume prune -a：**删除所有没有任何容器在使用的卷

**小结：**

docker run = 使用镜像创建+运行容器

每次执行都会创建一个新的容器

**六、docker run的其他参数**

**docker run -e xxx：**传递环境变量，可多次-e

- 可在dockerhub上搜索容器对应镜像，可传递的环境变量有哪些

**docker run** **--****name 容器名：**指定容器的名字，该名字在整个宿主机上必须唯一，不能重复

**docker run -it：**控制台可以进入容器内部交互，类似于从宿主机进入了容器的cmd

**docker run --rm：**当容器停止时，删除容器

- -it 常常和 --rm搭配使用，用于临时调试容器

**docker run --restart xxx：**用于配置容器停止时的重启策略

- always：容器停止就立即重启
- unless-stopped：意外停止重启，但手动停止不重启

**docker rm -f 容器ID或容器名**

- 删除容器
- -f：force，强制删除，对正在运行的容器需要加上



**七、调试容器**

如果不想创建容器，只想对已有容器进行启停，使用stop/start

- docker stop 容器ID或容器名：停止一个正在运行的容器，停止后，docker ps查询不到该容器
- docker start 容器ID或容器名：重新启动一个已停止的容器。
- 使用stop/start重新启停容器时，不需要再写已写过的端口映射、挂载卷和环境变量，docker已自动记录并保存，可通过docker inspect 容器ID或容器名查看，让AI分析输出结果

**docker create：**使用和docker run相同，但只创建容器，不立即启动（回顾：docker run = 创建+启动），如果要启动，用 docker start

**docker logs 容器ID或容器名**：查看容器的命令

- -f: follow 滚动查看日志，实时刷新。

**Docker技术原理简述**

利用Linux内核的两大原生功能实现容器化

- Cgroups (Control Groups): 用于限制和隔离进程的资源使用（为每个容器设置CPU、内存、网络带宽等），确保容器资源消耗不影响宿主机或其他容器。
- Namespaces: 用于隔离进程的资源视图，使得容器只能看到自己内部的进程ID、网络资源和文件目录，而看不到宿主机的。
- 本质: Docker容器本质上是一个特殊的进程，但进入容器内部后，其表现看起来像一个独立的操作系统。每个docker容器都是一个独立的运行环境，每个容器内部表现的都像一个独立的Linux系统

**docker exec** **容器ID或容器名** **Linux命令**：在一个正在运行的Docker容器内部执行Linux命令。

- 示例: docker exec my_nginx ps -ef 查看容器内进程。由于资源视图隔离，看不见宿主机进程。

**docker exec -it 容器ID /bin/sh**：进入容器内部获得交互式命令行环境，可进行文件系统查看、进程管理或深入调试。

- 可以直接在命令行环境里执行cd、ls等
- docker为尽量压缩镜像的大小，容器内部通常是极简操作系统，可能缺失vi等常用工具，需要自行安装：参考24:18



**八、构建镜像**

dockerfile：是一个文本文件，详细列出了如何制作Docker镜像的步骤和指令

- dockerfile：图纸
- 镜像：模具
- 容器：糕点

使用dockerfile制作一个镜像并推送到docker hub上（以将一个python程序打包为docker镜像为例）：25:43

- **准备Dockerfile文件**（这段好多看不懂啊）
- FROM 基础镜像: 所有Dockerfile的第一行，选择一个基础镜像，表示新镜像在此基础上构建。
- WORKDIR 目录路径: 设置镜像内的工作目录，后续命令在此目录下执行。
- COPY 源路径 目标路径: 将宿主机的文件或目录拷贝到镜像内的指定路径。
- RUN 命令: 在镜像构建过程中执行的命令（例如安装依赖）。
- EXPOSE 端口号: 声明镜像提供服务的端口（仅为声明，非强制，实际端口映射仍由`-p`参数决定）。
- CMD 命令: 容器运行时默认执行的启动命令。一个Dockerfile只能有一个CMD指令。
- ENTRYPOINT 命令: 与CMD类似，但优先级更高，不易被docker run命令覆盖。
- **构建镜像：**docker build -t 镜像名称:版本号 Dockerfile所在目录。
- docker build -t docker-test **.** (在当前目录构建名为`docker-test`的镜像)。
- 镜像构建好之后，可以使用docker run基于该镜像创建一个容器
- **将镜像推送至Docker Hub**
- docker login
- 构建镜像：要带上namespace【docker build -t <用户名>/<镜像名称>[:<版本号>]】。推送时镜像名称必须包含用户名作为命名空间。
- docker push <用户名>/<镜像名称>[:<版本号>]
- **在docker hub上用<用户名>/<镜像名称>搜索镜像**



**九、docker网络**

**1、桥接模式**

Docker网络默认是Bridge (桥接模式)，所有容器默认连接到此网络。每个容器被分配一个内部IP地址（通常是`172.17.x.x`开头）。

在内部子网中，**容器可以通过内部IP地址互相访问。但容器网络与宿主机网络隔离**，需通过端口映射（`-p`）才能从宿主机访问。

**创建子网：docker network create <子网名称>**

- 该子网默认也是桥接模式

**指定容器加入不同的子网：**docker run -d --name <容器名称> --network <子网名称> 

- 同一个子网的容器可以互相通信，而跨子网则不行。
- 同一子网内的容器可以使用**容器名称**互相访问，而不必使用内部IP地址（Docker内部DNS机制可以把容器名字转换成IP地址）：
- docker run -e 加入环境变量时，可以使用容器名而不是IP地址。
- ping容器名字

**2、Host (主机模式)**

Docker容器直接共享宿主机的网络命名空间、 直接使用宿主机的IP地址，无需端口映射（`-p`参数），容器内的服务直接运行在宿主机的端口上，通过宿主机的IP和端口即可访问容器。

-  语法：docker run --network host
-  在浏览器直接访问服务器的IP地址加端口80就可以访问到容器

在容器内部查询ip地址：32:40

- 安装工具iproute2
- 命令：ip addr show
- 可以发现容器的内网地址=云服务器内网地址，说明容器共享了宿主机的网络空间

**3、None (不联网模式)**

容器不连接任何网络，完全隔离。

docker network list：展示出所有网络

- NAME中的bridge、host、none是三种默认网络，不能删除
- docker network rm <网络ID>= docker network remove <网络ID>：删除自定义子网



十、Docker Compose（多容器编排）

当一个完整的应用由多个模块（如前端、后端、数据库）组成时

- 若将所有模块打包成一个巨大容器，会导致故障蔓延、可伸缩性差、扩容只能将整个大容器复制一份，做不到针对某个模块的精准性扩容。
- 最佳实践是把若每个模块独立容器化，但是管理多个容器（创建、网络配置）会增加使用成本（多次执行docker run并需要配置容器间网络）——引入容器编排技术。

**docker compose 使用 yml文件管理多个容器**

- yml文件列出了容器如何创建、如何协同工作的说明。可理解为多个docker run命令
- docker run命令和docker compose文件的对比关系

services: 顶级元素，每个服务对应一个容器。

image: 对应docker run中的镜像名。

environment`: 对应docker run的-e参数。

volumes: 对应docker run的-v参数（挂载卷）。

ports: 对应docker run的-p参数。

网络：Docker Compose会自动为每个Compose文件创建一个默认子网，文件中定义的所有容器都会自动加入此子网，并可通过服务名称互相访问。不像docker run命令那样需要显式地为容器指定子网。

**docker compose可以自定义容器的启动顺序，确保依赖服务先启动**

- depends_on

AI辅助：可借助AI工具生成等价的Docker Compose文件。

- 创建docker compose文件：36:19
- docker compose up: 启动YAML文件中定义的所有服务（容器）。
- 会自动创建子网和容器。
- -d: 后台运行。
- docker ps检查正在运行的容器
- 如果容器已经在运行，再执行一次docker compose up并不会启动新的容器，没有任何效果
- docker compose down: 停止并删除由Compose文件定义的所有服务和网络。
- docker ps -a
- docker compose stop：仅停止服务，不删除容器。
- docker compose start：启动已停止的服务。
- docker compose -f </目录/文件名.yml> up: 指定非标准文件名（docker-compose.yaml）的Compose文件进行操作。

小结：

Docker Compose是一个轻量级的容器编排技术，适合个人使用和单机运行。

企业级服务器集群大规模容器编排：可以使用软件Kubernetes



十一、总结

1、docker的核心概念：容器、镜像、镜像仓库

2、docker的安装方法

3、下载镜像，配置镜像站（上传镜像）的方法

4、使用docker run创建、运行容器，及重要参数

- -p：端口映射
- -v：挂载卷
- -e：环境变量

5、进入容器内部进行调试

6、docker网络：bridge、host、none，如何创建子网

7、dockerfile

8、docker compose