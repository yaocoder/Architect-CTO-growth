编写Dockerfile的常用命令
=============

### 一、命令简介
| 命令 | 描述           |
|--------|---------------|
| FROM   | 基于哪个镜像来实现 | 
| MAINTAINER | 镜像创建者 |
| ENV | 声明环境变量 |
| RUN | 执行命令 |
| ADD | 添加宿主机文件到容器里，有需要解压的文件会自动解压 |
| COPY | 添加宿主机文件到容器里 | 
| WORKDIR | 工作目录 |
| EXPOSE | 容器内应用可使用的端口 |
| CMD |容器启动时执行的命令，如果执行 docker run <启动命令> CMD 会被覆盖|
| ENTRYPOINT |与 CMD 功能基本等同，但执行 docker run <启动命令> 不会覆盖，如果需要覆盖可以增加参数 -entrypoint|
| VOLUME | 数据卷，将宿主机的目录映射到容器中的目录 | 

### 二、命令详解
#### FROM：
指定基础镜像，所有构建的镜像都必须有一个基础镜像，且 FROM 命令必须是 Dockerfile 的第一个命令
```
FROM <image> [AS <name>] 指定从一个镜像构建起一个新的镜像名字
FROM <image>[:<tag>] [AS <name>] 指定镜像的版本 Tag
示例：FROM mysql:5.0 AS database
```

#### MAINTAINER：
镜像维护人的信息
```
MAINTAINER <name>
示例：MAINTAINER dayao yaocoder@gmail.com
```

#### RUN：
构建镜像时要执行的命令
```
RUN <command>
示例：RUN [executable, param1, param2]
```

#### ADD：
将本地的文件添加复制到容器中去，压缩包会解压，可以访问网络上的文件，会自动下载
```
ADD <src> <dest>
示例：ADD *.py /app 添加 python 文件到容器中的 app 目录下
```

#### COPY：
功能和 ADD 基本等同，只是复制，不会解压或者下载文件
```
COPY <src> <dest>
示例：COPY *.py /app 拷贝 python 文件到容器中的 app 目录下
```

#### CMD：
启动容器后执行的命令，如果执行 docker run <启动命令> CMD 会被覆盖
```
示例：CMD [executable, param1, param2]
CMD [ "python3", "./helloworld.py" ]
执行 python3 ./helloworld.py
```

#### ENTRYPOINT：
执行命令，和 CMD 基本一样，但执行 docker run <启动命令> 不会覆盖，如果需要覆盖可以增加参数 -entrypoint
```
ENTRYPOINT [executable, param1, param2]
示例：ENTRYPOINT [python3, ./helloworld.py]
```

#### LABEL：
为镜像添加元数据，key-value 形式
```
LABEL <key>=<value> <key>=<value> ...
示例：LABEL version=1.0 description= webServer
```

#### ENV：
设置环境变量，有些容器运行时会需要某些环境变量
```
ENV <key> <value> 一次设置一个环境变量
ENV <key>=<value> <key>=<value> <key>=<value> 设置多个环境变量
示例：ENV JAVA_HOME /usr/java1.8/
```

#### EXPOSE：
暴露容器的端口（容器内部程序的端口，需要将其映射到一个外部访问端口）
```
EXPOSE <port>
示例：EXPOSE 80
容器运行时，需要用 -p 映射外部端口才能访问到容器内的端口
```

#### VOLUME：
挂载卷，指定数据持久化的目录
```
VOLUME /var/log 
指定容器中需要被挂载的目录，会把这个目录映射到宿主机的一个随机目录上，实现数据的持久化和同步
VOLUME [/var/log,/var/test.....] 
指定容器中多个需要被挂载的目录，会把这些目录映射到宿主机的多个随机目录上，实现数据的持久化和同步
VOLUME /var/data var/log 
指定容器中的 var/log 目录挂载到宿主机上的 /var/data 目录，这种形式可以手动指定宿主机上的目录
```

#### WORKDIR：
设置工作目录，设置之后 ，RUN、CMD、COPY、ADD 的工作目录都会同步变更
```
WORKDIR <path>
示例：WORKDIR /usr/src/app
```

#### USER：
指定运行命令时所使用的用户，为了安全和权限起见，根据要执行的命令选择不同用户
```
USER <user>:[<group>]
示例：USER dayao
```

#### ARG：
设置构建镜像是要传递的参数
```
ARG <name>[=<value>]
ARG name=dayao
```

   