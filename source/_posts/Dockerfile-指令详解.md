---
title: Dockerfile 指令详解
date: 2021-06-04 17:50:03
tags: docker,容器
---

### COPY 复制文件
```bash
COPY <源路径>... <目标路径>
COPY ["<源路径1>",... "<目标路径>"]
```
> - 和 RUN 指令一样，也有两种格式，一种类似于命令行，一种类似于函数调用。
COPY 指令将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置。比如：
COPY package.json /usr/src/app/

> - <源路径> 可以是多个，甚至可以是通配符，其通配符规则要满足 Go 的 filepath.Match 规则，如：
COPY hom* /mydir/
COPY hom?.txt /mydir/

> - <目标路径> 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径（工作目录可以用 WORKDIR指令来指定）。目标路径不需要事先创建，如果目录不存在会在复制文件前先行创建缺失目录。

> - 此外，还需要注意一点，使用COPY指令，源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。这个特性对于镜像定制很有用。特别是构建相关文件都在使用 Git 进行管理的时候.

### ADD 更高级的复制文件

> ADD 指令和 COPY 的格式和性质基本一致。但是在 COPY 基础上增加了一些功能。

#### 源路径可以是URL

> 比如 <源路径> 可以是一个 URL，这种情况下，Docker 引擎会试图去下载这个链接的文件放到 <目标路径> 去。下载后的文件权限自动设置为 600，如果这并不是想要的权限，那么还需要增加额外的一层 RUN进行权限调整

#### 源文件可以是压缩包

> 另外，如果下载的是个压缩包，需要解压缩，也一样还需要额外的一层 RUN 指令进行解压缩。所以不如直接使用 RUN 指令，然后使用 wget 或者 curl工具下载，处理权限、解压缩、然后清理无用文件更合理。因此，这个功能其实并不实用，而且``不推荐使用``。
如果 <源路径> 为一个 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，ADD 指令将会自动解压缩这个压缩文件到 <目标路径> 去。
在某些情况下，这个自动解压缩的功能非常有用，比如官方镜像 ubuntu 中：
```bash
FROM scratch
ADD ubuntu-xenial-core-cloudimg-amd64-root.tar.gz /
...
```

#### COPY和ADD适用场景说明

> - 在 Docker 官方的最佳实践文档中要求，尽可能的使用 COPY，因为 COPY 的语义很明确，就是复制文件而已，而 ADD 则包含了更复杂的功能，其行为也不一定很清晰。最适合使用 ADD 的场合，就是所提及的需要自动解压缩的场合。
> - 另外需要注意的是，ADD 指令会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢。
因此在 COPY 和 ADD 指令中选择的时候，可以遵循这样的原则，所有的文件复制均使用 COPY 指令，仅在需要自动解压缩的场合使用 ADD。

### CMD 容器启动命令
```bash
shell 格式：CMD <命令>
exec 格式：CMD ["可执行文件", "参数1", "参数2"...]
参数列表格式：CMD ["参数1", "参数2"...]。在指定了 ENTRYPOINT 指令后，用 CMD 指定具体的参数。
```

> - Docker 不是虚拟机，容器就是进程。既然是进程，那么在启动容器的时候，需要指定所运行的程序及参数。CMD 指令就是用于指定默认的容器主进程的启动命令的。
> - 在运行时可以指定新的命令来替代镜像设置中的这个默认命令，比如，ubuntu 镜像默认的 CMD 是`/bin/bash`，如果我们直接 `docker run -it ubuntu` 的话，会直接进入bash。我们也可以在运行时指定运行别的命令，如 `docker run -it ubuntu cat /etc/os-release`。这就是用 `cat /etc/os-release` 命令替换了默认的 `/bin/bash` 命令了，输出了系统版本信息。
> - 在指令格式上，一般推荐使用 `exec` 格式，这类格式在解析时会被解析为 JSON 数组，因此一定要使用双引号 "，而不要使用单引号。
> - 如果使用 shell 格式的话，实际的命令会被包装为 sh -c 的参数的形式进行执行。比如：`CMD echo $HOME`在实际执行中，会将其变更为：`CMD [ "sh", "-c", "echo $HOME" ]`
> - 这就是为什么我们可以使用环境变量的原因，因为这些环境变量会被 shell 进行解析处理。
提到 CMD 就不得不提容器中应用在前台执行和后台执行的问题。这是初学者常出现的一个混淆。
> - Docker 不是虚拟机，容器中的应用都应该以前台执行，而不是像虚拟机、物理机里面那样，用 upstart/systemd 去启动后台服务，容器内没有后台服务的概念。一些初学者将 CMD 写为：`CMD service nginx start`
> - 然后发现容器执行后就立即退出了。甚至在容器内去使用systemctl命令结果却发现根本执行不了。这就是因为没有搞明白前台、后台的概念，没有区分容器和虚拟机的差异，依旧在以传统虚拟机的角度去理解容器。
> - 对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。
> - 而使用 service nginx start 命令，则是希望 upstart 来以后台守护进程形式启动 nginx 服务。而刚才说了 CMD service nginx start 会被理解为 CMD [ "sh", "-c", "service nginx start"]，因此主进程实际上是 sh。那么当 service nginx start 命令结束后，sh 也就结束了，sh 作为主进程退出了，自然就会令容器退出。
> - 正确的做法是直接执行 nginx 可执行文件，并且要求以前台形式运行。比如：CMD ["nginx", "-g", "daemon off;"]

#### ENTRYPOINT 入口点
> 让镜像变成像命令一样使用;应用运行前的准备工作

### ENV 设置环境变量
```bash
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...
```

> 这个指令很简单，就是设置环境变量而已，无论是后面的其它指令，如RUN，还是运行时的应用，都可以直接使用这里定义的环境变量。

### ARG 构建参数
```bash
ARG <参数名>[=<默认值>]
```

> 构建参数和 ENV 的效果一样，都是设置环境变量。所不同的是，ARG所设置的构建环境的环境变量，在将来容器运行时是不会存在这些环境变量的。但是不要因此就使用 ARG 保存密码之类的信息，因为 docker history 还是可以看到所有值的。


### VOLUME 定义匿名卷
```bash
VOLUME ["<路径1>", "<路径2>"...]
VOLUME <路径>
```

### EXPOSE 声明端口
```bash
EXPOSE <端口1> [<端口2>...]。
```

> EXPOSE 指令是声明运行时容器提供服务端口，这只是一个声明，在运行时并不会因为这个声明应用就会开启这个端口的服务。

### WORKDIR 指定工作目录
```bash
WORKDIR <工作目录路径>。
```

> 使用 WORKDIR 指令可以来指定工作目录（或者称为当前目录），以后各层的当前目录就被改为指定的目录，如该目录不存在，WORKDIR 会帮你建立目录。

### USER 指定当前用户
```bash
USER <用户名>
```

> USER 指令和 WORKDIR 相似，都是改变环境状态并影响以后的层。WORKDIR是改变工作目录，USER则是改变之后层的执行 RUN, CMD 以及 ENTRYPOINT 这类命令的身份。

### HEALTHCHECK 健康检查
```bash
HEALTHCHECK [选项] CMD <命令>：设置检查容器健康状况的命令
HEALTHCHECK NONE：如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令
```

### ONBUILD 为他人做嫁衣裳
```bash
ONBUILD <其它指令>。
```
> ONBUILD 是一个特殊的指令，它后面跟的是其它指令，比如 RUN, COPY等，而这些指令，在当前镜像构建时并不会被执行。只有当以当前镜像为基础镜像，去构建下一级镜像的时候才会被执行。
