#Docker踩坑小记

##1. 基本概念及命令
------------------
docker官网：docker.com
可以按照**[Get Started](http://docs.docker.com/mac/started/)**流程针对各个平台进行安装和初步入门。
基本概念和知识就不赘述了，可以直接看官网的docker doc or [dockerpool.com](http://dockerpool.com/static/books/docker_practice/introduction/what.html)上的翻译

####镜像(image)
一个镜像文件类似.iso,是一个完整的操作环境。每个镜像都配有一个标签tag，用于标注image的版本。

常用命令:

+ docker images 查看本地所有镜像

+ docker tag IMAGE[:TAG] [REGISTRYHOST/][USERNAME/]NAME[:TAG]
	
+ docker pull IMAGE_NAME[;TAG] （若本地没有则直接从docker hub下载, eg. docker pull ubuntu）
	
+ docker push NAME[:TAG] push本地镜像到docker hub上自己账户的对应仓库中，前提是账号中有该仓库。

+ docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
	
		常用OPTIONS:

		-i -t，一般同时用-it，用于与容器交互，i:interactive, t:分配一个伪终端名
		
		-p OUT-PORT:INNER-PORT, 设置端口转换。

			没有的话会随机设置一个端口，需要通过docker inspect CONTAINER查看容器设置

		-v OUT-PATH:INNER-PATH,挂载数据卷，将宿主机某个目录拷贝到容器内某个目录。
			
			原因在于docker镜像只会保存环境的设置并不会保存数据.
			
			如果冒号后面没有，docker会自动分配一个临时目录用于挂载，

			docker inspect -f {{.Volumes}} CONTAINER_NAME 查看
		
		--name CONTAINER_NAME， 设置容器名称
		
		-d, 是否在后台运行
	
+ docker rmi IMAGE, 删除镜像，如果输入的是REPOSITORY下的名称，只会Untag 该仓库和对应IMAGE的关系，因为同一个image可能会关联到不同仓库


####容器(container)

docker中用于运行镜像的标准单位，用户可以按照自己的需求一层层网上搭。比如一个空Unbuntu环境的container中加入mysql, apache等其他镜像文件构成一个大的container，这是通过Dockerfile文件实现，下面会叙述。
	
常用命令: 

+ docker ps, 查看正在运行的容器

+ docker ps -a, 查看所有容器包括没有运行的

+ docker start CONTAINER ID/NAME, 启动容器

+ docker stop CONTAINER ID/NAME, 停止容器
			 
+ docker rm CONTAINER ID/NAME, 删除容器
			 
+ docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]], 提交修改过的container中新镜像，eg.start一个空ubuntu镜像在其中安装了mysql安装包即可以将新环境提交为一个新的镜像
			 
+ docker exec [OPTIONS] CONTAINER COMMAND [ARG...], 类似run命令，对running container进行操作

####仓库(hub)

存放镜像文件的场所，分public和private。Docker Hub是最大的镜像仓库
	
+ public: 大部分镜像都是存放在docker hub public, 但由于在墙外，所以国内下载会异常慢，有两种解决方法：1.找一个国内[public hub](https://www.daocloud.io/)下载,可以进行下载加速。2.自己搭建一个docker registry service，下面会讲。
	
+ private: 类似github你可以在docker hub上建一个private hub用于存放你的镜像。然后你可以搭建本地私有镜像库
	
+ docker-registry: 你可以通过官方提供的docker-regitry在本地搭载一个private hub将image存储在本地，registry默认端口5000，所有你以后tag, push, pull的地址都是docker ip:5000。当然你可以使用云存储amazon s3 or aliyun oss服务，这边就牵扯到了服务驱动的问题。在国内一般使用aliyun oss服务，github上有人写了一个[oss driver](https://github.com/chris-jin/docker-registry-driver-alioss.git),也是阿里云官方推荐的。操作指南可参考README or 附件《阿里云 DOCKER 实践》or [blog](http://blog.after1980.com/?p=602)

####Dockerfile
构建镜像的命令文件，image可以直接下，也可以通过自己编写Dockerfile构建

构建镜像通常有两种方式

+ 使用Dockerfile写入命令直接build镜像，但这个需要你事先对所需要安装的软件包以及环境配置非常清楚才能提前写好。

+ 比较繁琐但更加容易上手，比如：run一个空ubuntu镜像，通过`docker exec -it CONTAINER_NAME /bin/bash`命令进入container内部一个个自己安装，然后直接commit成一个新的镜像。
	
常用命令: 

+ docker build [OPTIONS] PATH | URL | - , docker会查找指定目录下的Dockerfile文件以及所有内容发给服务器由服务器来构建镜像,所以一般建议放置Dockerfile的为空目录


> 坑1：docker在Linux,Centos,Ubuntu等os上是直接以service形式运行，然而在mac和windows上是基于*VM*上运行，相当于是一个大虚拟机里嵌入了一个小虚拟机。
  所以这边需要注意的在于网上所有教程都告诉你运行container之后直接输入网址localhost or 127.0.0.1就可以看到页面，但这只适用于Linux,Ubuntu等，mac 和 windows并不适用。因为本质你是要访问的docker service的ip，那么在mac和windows上因为是基于vm的，所以你要访问的是vm ip.可以通过*`boot2docker ip`*命令查看。通常是192.168.59.103。
  那么基于上述原因，在service端两者是截然不同的，mac and windows下你需要输入boot2docker ssh进入虚拟机才能操作docker service。即在Linux, ubuntu, centos下，只有两层，docker service and client都在宿主机上，container直接运行在宿主机上，而在mac and windows下，一共有三层，宿主机-vm-container，client在宿主机，service在vm，一切对docker的操作都基于vm。

> 坑2：mac or windows在启动Boot2docker时一般会报错：x509: certificate is valid for 127.0.0.1, 10.0.2.15, not 192.168.59.103。
  原因同上，docker service ip并不是localhost所以会报证书错。直接运行"boot2docker ssh sudo /etc/init.d/docker restart"重启即可。

> 坑3：如果你使用阿里云docker镜像库，你可能会碰到如下报错信息
	Error: Invalid registry endpoint https://registry.mirrors.aliyuncs.com/v1/:.......connection refused. If this private registry supports only HTTP or HTTPS with an unknown CA certificate, please add `--insecure-registry registry.mirrors.aliyuncs.com` to the daemon's arguments. In the case of HTTPS, if you have access to the registry's CA certificate, no need for the flag; simply place the CA certificate at /etc/docker/certs.d/registry.mirrors.aliyuncs.com/ca.crt
	正如报错信息显示，是因为你在使用docker hub以外的其他镜像库而产生的证书安全问题，只需要按照提示把"--insecure-registry registry-url"加入到配置文件中即可。在linux下docker文件在/etc/default/docker，在centos下，文件在/etc/sysconfig/docker，加入“other_args="--insecure-registry=registry.mirrors.aliyuncs.com"”即可。
	需要记住的是在不同操作系统下docker的配置及路径不一定相同，所以在查资料的时候需要留意自己的系统是否和别人相同

> 坑4: mac or windows上使用boot2docker服务时会牵扯到3个环境变量分别是DOCKER_HOST即上文所说的vm ip, DOCKER_CERT_PATH, DOCKER_TLS_VERIFY。
  你可以在boot2docker中输入env or boot2docker shellinit查看当前环境变量中是否包含这3个变量，如果没有，后者会提醒你怎样设置 eg. export DOCKER_HOST=xxxx. 也可以运行 eval "$(boot2docker shellinit)" 进行自动设置。想验证是否正常运行，输入boot2docker status查看即可。

> 坑5: 在大部分报错情况中，你遇到的都会是“ Cannot connect to the Docker daemon. Is 'docker -d' running on this host?”，但原因却各种各样，例如: 你系统内核版本不够, docker ip被占了service没跑起来(这个情况在ECS上可能会出现，参考[15L](http://bbs.aliyun.com/read/152090.html), docker脑抽重启几次就好。

> 坑6: 在输入命令并没有得到相应结果的情况时，在排除配置bug或输入bug以外，尝试多运行几次，尤其是在docker pull时

> 坑7: 网上很多回答docker相关问题下都会有让你直接`boot2docker init`什么什么的，个人强烈不建议这么多,因为这是直接初始化vm，你之前保存的所有镜像和配置都会没有，这对于程序猿的工作是毁灭性的= =，所以除非万不得已不要初始化，并且我目前遇到的所有问题都是可以通过对应方法解决的，并不需要这样万金油的措施

> 坑7: 关于镜像更新，docker使用tag, pull, push对镜像标签，拉去，提交，**但千万不要以为pull就直接更新了**，并不会。当某个镜像其他人更新提交后你需要做的是删除本地的该镜像，然后重新pull，就能得到最新的镜像。

##2.volumn数据卷
------------------

概念很简单，就是把宿主机的某个目录挂载到container中系统下的某个目录，并且文件可以实时更新。数据卷和数据卷容器使用起来很简单，备份恢复迁移直接按文档走一遍即可，一般遇到的都是权限问题，而你首先需要了解的是为什么要使用Volumn。

Volumn一方面可以持久化数据，一方面可以将容器与数据分离。举个简单的例子，你运行了一个搭载mysql image的container A，然后进行该数据库，在其中进行数据操作比如添加一个数据库，添加个表，添加一行数据等等。然后你retart container,进入container查看发现那些库表数据还在，因为它保存在容器中而容器并没有销毁。现在你commit container A 为一个新的镜像mysql-1并且rm container A,然后重新运行一个搭载mysql-1 image的container B，你会发现你之前操作的所有数据都不见了。在不使用Volumn的情况下，数据是跟着容器走的，容器在，数据就在，容器不在，数据就不在，即使你提交为一个新的镜像mysql-1，它也和mysql没有区别.

需要记住的是docker本质是帮助开发者统一不同场景下的操作环境和配置文件，而不是帮你保存数据。

> 坑1: mac环境下，使用Volumn保存mysql数据文件时一直会遇到权限问题，即在-v设置宿主机上mysql数据保存目录时，container中的user mysql并没有权限对宿主机上的目录进行写操作。可以把该目录设置为任何人都有读写权限。

> 坑2: 在使用Volumn保存服务器文件情况下，更改html or js会直接更新，但是更改后台文件比如.py，网页并不会更新，原因在于container中的apache使用的时已经编译过的.pyc文件且有缓存，所以并不会实时更新后台文件，需要开发人员在外部使用docker exec命令重启apache, eg. docker exec -it /etc/init.d/apache2 restart.


##3. compose
------------------
compose可以帮助开发人员通过一个.yml文件来定义一组相关联的容器组成一个集群服务，比如一个web服务容器+一个后端数据容器。

这边并没有什么坑，照着官网文档做就好。

> Docker Machine, Docker Swarm以及Docker自动化构建等用到的时候再更新吧= =


