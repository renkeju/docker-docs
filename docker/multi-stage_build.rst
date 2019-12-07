多阶段构建
~~~~~~~~~~~~~~

之前的做法
^^^^^^^^^^^^^^^^

在 **Docker 17.05** 版本之前，我们构建 Docker 镜像时，通产会采用两种方式：

* 全部放入一个 Dockerfile

    一种方式是将所有的构建过程包含在一个 :guilabel:`Dockerfile` 中，包括项目及其依赖库的编译、测试、打包等流程，这里可能会带来一些问题：

    * :guilabel:`Dockerfile` 特别长，可维护性降低
    * 镜像层次多，镜像体积大较大，部署时间长
    * 源代码存在泄漏的风险

    例如，编写 ``app.go`` 文件，该程序输出 ``Hello World!``

    .. code-block:: none

        package main

        import "fmt"

        func main() {
            fmt.Printf("Hello World!");
        }

    编写 ``Dockerfile.one`` 文件

    .. code-block:: none

        FROM golang:1.9-alpine

        RUN apk --no-cache add git ca-certificates

        WORKDIR /go/src/github.com/go/hellworld/

        COPY app.go .

        RUN go get -d -v github.com/go-sql-driver/mysql \
            && CGO_ENABLED=0 GOOS=linux go build -a installsuffix cgo -o app . \
            && cp /go/src/github.com/go/helloworld/app /root

        WORKDIR /root/

        CMD ["./app"]

    构建镜像

    .. code-block:: none

        $ docker image build --tag go/helloworld:1 --file Dockerfile.one .

* 分散到多个 Dockerfile

    另一种方式，就是我们事先在一个 :guilabel:`Dockerfile` 将项目及其依赖库编译测试打包好后，再将其拷贝到运行环境中，这种方式需要我们编写两个 :guilabel:`Dockerfile` 和一些编译脚本才能将其两个阶段自动整合起来，这种方式虽然可以很好地规避第一种方式存在的风险，但明显部署过程较复杂。

    例如 
    
    编写 ``Dockerfile.build`` 文件

    .. code-block:: none

        FROM golang:1.9-alpine

        RUN apk --no-cache add git

        WORKDIR /go/src/github.com/go/helloworld

        COPY app.go .

        RUN go get -d -v github.com/go-sql-driver/mysql \
            && CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

    编写 ``Dockerfile.copy`` 文件

    .. code-block:: none

        FROM alpine:latest

        RUN apk --no-cache add ca-certificates

        WORKDIR /root/

        COPY app .

        CMD ["./app"]

    新建 ``build.sh``

    .. code-block:: none

        #!/bin/sh
        echo Building go/helloworld:build

        docker image build --tag go/helloworld:build . --file Dockerfile.build

        docker container create --name extract go/helloworld:build
        docker container cp extract:/go/src/github.com/go/helloworld/app ./app
        docker container rm --force extract

        echo Building go/helloworld:2

        docker image build --no-cache --tag go/helloworld:2 . --file Dockerfile.copy
        rm ./app

    现在运行脚本即可构建镜像

    .. code-block:: none

        $ chmod +x build.sh
        $ ./build.sh

    对别两种方式生成的镜像大小

    .. code-block:: none 

        $ docker image ls
        REPOSITORY      TAG IMAGE           ID  CREATED     SIZE
        go/helloworld   2   f7cf3465432c    22  seconds ago 6.47MB
        go/helloworld   1   f55d3e16affc    2   minutes ago 295MB

使用多阶段构建
^^^^^^^^^^^^^^^^^^^

为解以上的问题，**Docker v17.05** 开始支持多阶段构建（``multistage builds``）。使用多阶段构建我们就可以很容易解决前面提到的问题，并且只需要编写一个 :guilabel:`Dockerfile` ：

例如

编写 :guilabel:`Dockerfile` 文件

.. code-block:: none

    FROM golang:1.9-alpine as builder

    RUN apk --no-cache add git

    WORKDIR /go/src/github.com/go/helloworld/

    RUN go get -d -v github.com/go-sql-driver/mysql

    COPY app.go .

    RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

    FROM alpine:latest as prod

    RUN apk --no-cache add ca-certificates

    WORKDIR /root/

    COPY --from=0 /go/src/github.com/go/helloworld/app .

    CMD ["./app"]

构建镜像

.. code-block:: none

    $ docker image build --tag go/helloworld:3 .

对比三个镜像的大小

.. code-block:: none

    $ docker image ls
    REPOSITORY      TAG IMAGE           ID              CREATED SIZE
    go/helloworld   3   d6911ed9c846    7 seconds ago   6.47 MB
    go/helloworld   2   f7cf3465432c    22 seconds ago  6.47 MB
    go/helloworld   1   f55d3e16affc    2 minutes ago   295 MB

很明显使用多阶段构建的镜像体积小，同时也完美解决了上边提到的问题。

只构建某一阶段的镜像
^^^^^^^^^^^^^^^^^^^^^^^^^^

我们可以使用 ``as`` 来为某一阶段命令，例如：

.. code-block:: none

    FROM golang:1.9-alpine as builder

例如当我们只构建 ``builder`` 阶段的镜像时，我们可以使用 ``docker image build`` 命令时加上 ``--target`` 参数即可

.. code-block:: none

    $ docker image build --target builder --tag username/imagename:tag

构建时聪其他镜像复制文件
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

上面例子中我们使用 ``COPY --from=0 /go/src/github.com/go/helloworld/app .`` 从上一阶段的镜像中复制文件，我们也可以复制任何镜像中的文件。

.. code-block:: none

    COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf