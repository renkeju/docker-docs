ENTRYPOINT 入口点
^^^^^^^^^^^^^^^^^^^^^^

:guilabel:`ENTRYPOINT` 的格式和 :guilabel:`RUN` 指令格式一样，分别为 **exec** 格式和 **shell** 格式。

:guilabel:`ENTRYPOINT` 的目的和 :guilabel:`CMD` 一样，都是在指定容器启动程序及参数。:guilabel:`ENTRYPOINT` 在运行时也可以替代，不过比 :guilabel:`CMD` 要略显繁琐，需要通过 ``docker container run`` 的参数 ``--entrypoint`` 来指定。

当指定了 :guilabel:`ENTRYPOINT` 后，:guilabel:`CMD` 的含义就发生了改变，不再是直接的运行其命令，而是将 :guilabel:`CMD` 的内容作为参数传给 :guilabel:`ENTRYPOINT` 指令，换句话说实际执行时，将变为：

.. code-block:: none

    <ENTRYPOINT> "<CMD>"

那么有了 :guilabel:`CMD` 后，为什么还要有 :guilabel:`ENTRYPOINT` 呢？这种 ``<ENTRTYPOINT> <CMD>`` 有什么好处？让我们来看几个场景。

* 场景一：让镜像变成像命令一样使用

    假设我们需要一个得知自己当前公网IP的镜像，那么可以先用 :guilabel:`CMD` 来实现：

    .. code-block:: none

        FROM ubuntu:16.04
        RUN apt-get update \
            && apt-get install -y curl \
            && rm -rf /var/lib/apt/lists/*
        CMD ["curl", "-s", "http://ip.cn"]

    假如我们使用 ``docker image build --tag=myip .`` 来构建镜像的话，如果我们需要查询当前公网IP，只需要执行：

    .. code-block:: none

        docker container run --rm --name myip_test myip:1.0
        当前 IP: 58.246.147.26 来自: 上海市 联通

    嗯，这么看起来我们好像直接把镜像当作命令使用了，不过命令总有参数，如果我们希望加参数呢？比如从上面的 :guilabel:`CMD` 中可以看到实质的命令时 curl，那么如果我们希望显示 HTTP 头信息，就需要加上 ``-i`` 参数。那么我们可以直接加入 ``-i`` 参数给 ``docker container run myip`` 吗？

    .. code-block:: none

        docker container run --rm --name myip_test myip:1.0 -i
        docker: Error response from daemon: OCI runtime create failed: container_linux.go:348: starting container process caused "exec: \"-i\": executable file not found in $PATH": unknown.

    我们可以看到可执行文件找不到的报错，``executable file not found`` 。之前我们说过，跟在镜像名后面的是 command，运行时会替换 :guilabel:`CMD` 的默认值。因此这里的 ``-i`` 替换了原来的 :guilabel:`CMD` ，而不是添加在原来的 ``curl -s http://ip.cn`` 后面。而 ``-i`` 根本不是命令，所以自然找不到。

    那么如果我们希望加入 ``-i`` 这参数，我们就必须重新完整的输入这个命令：

    .. code-block:: none

        $ docker container run --rm --name myip_test myip:1.0 curl -s http://ip.cn -i
        HTTP/1.1 200 OK
        Date: Thu, 01 Nov 2018 02:59:52 GMT
        Content-Type: text/html; charset=UTF-8
        Transfer-Encoding: chunked
        Connection: keep-alive
        Set-Cookie: __cfduid=d8b77ba972fb91bec979f9a212ceca6841541041192; expires=Fri, 01-Nov-19 02:59:52 GMT; path=/; domain=.ip.cn; HttpOnly
        Server: cloudflare
        CF-RAY: 472b1b9af36b9619-SJC

        当前 IP: 58.246.147.26 来自: 上海市 联通

    这显然不是很好的解决方案，而使用 :guilabel:`ENTRYPOINT` 就可以解决这个问题。现在我们重新用 :guilabel:`ENTRYPOINT` 来实现这个镜像：

    .. code-block:: none

        FROM ubuntu:16.04

        RUN apt-get update \
            && apt-get install -y curl \
            && rm -rf /var/lib/apt/lists/*

        ENTRYPOINT [ "curl", "-s", "http://ip.cn" ]

    这次我们再来尝试直接使用 ``docker container run myip -i`` 。

    .. code-block:: none

        $ docker container run --rm --name myip_test myip:1.1
        当前 IP: 58.246.147.26 来自: 上海市 联通

        $ docker container run --rm --name myip_test myip:1.1 -i
        HTTP/1.1 200 OK
        Date: Thu, 01 Nov 2018 03:17:16 GMT
        Content-Type: text/html; charset=UTF-8
        Transfer-Encoding: chunked
        Connection: keep-alive
        Set-Cookie: __cfduid=d89a29d9467d6f00d7856b8a8f22d10791541042236; expires=Fri, 01-Nov-19 03:17:16 GMT; path=/; domain=.ip.cn; HttpOnly
        Server: cloudflare
        CF-RAY: 472b351850169668-SJC

        当前 IP: 58.246.147.26 来自: 上海市 联通

    可以看到，这次成功了。这是因为当存在 :guilabel:`ENTRYPOINT` 后，:guilabel:`CMD` 的内容将会作为参数传给 :guilabel:`ENTRYPOINT`，而这里 ``-i`` 就是新的 :guilabel:`CMD`，因为会作为参数传给 curl，从而达到了我们预期的效果。

* 场景二：应用运行前的准备工作

    启动容器就是启动主进程，但有些时候，启动主进程前，需要一些准备工作。

    比如 mysql 类的数据库，可能需要一些数据库配置、初始化的工作，这些工作要在最终的 mysql 服务器运行之前解决。

    此外，可能希望避免使用 :guilabel:`root` 用户去启动服务，从而提高安全性，而在启动服务前还需要以 root 身份执行一些必要的准备工作，最后切换到服务用户身份启动服务。或者除了服务之外，其他命令依旧可以使用 root 身份执行，方便调试等。

    这些准备工作是和容器 :guilabel:`CMD` 无关的，无论 :guilabel:`CMD` 是什么，都需要事先进行一个预处理工作。这种情况下，可以写一个脚本，然后放入 :guilabel:`ENTRYPOINT` 中执行，而这个脚本会将接收到的参数（也就是 :guilabel:`<CMD>`）作为命令，在脚本最后执行。比如官方镜像 `redis Dockerfile <https://github.com/docker-library/redis/blob/dc6dc737baa434528ce31948b22b4c6ccc78793a/5.0/Dockerfile>`_ 中就是这么做的：

    .. code-block:: none

        FROM alpine:3.4
        ...
        RUN addgroup -S redis && adduser -S -G redis redis
        ...
        ENTRYPOINT ["docker-entrypoint.sh"]

        EXPOSE 6379
        CMD [ "redis-server" ]

    可以看到其中为了 redis 服务创建了 redis 用户，并在最后指定了 :guilabel:`ENTRYPOINT` 为 ``docker-entrypoint.sh`` 脚本。

    .. code-block:: bash

        #!/bin/sh
        set -e

        # first arg is `-f` or `--some-option`
        # or first arg is `something.conf`
        if [ "${1#-}" != "$1" ] || [ "${1%.conf}" != "$1" ]; then
            set -- redis-server "$@"
        fi

        # allow the container to be started with `--user`
        if [ "$1" = 'redis-server' -a "$(id -u)" = '0' ]; then
            find . \! -user redis -exec chown redis '{}' +
            exec gosu redis "$0" "$@"
        fi

        exec "$@"

    该脚本的内容就是根据 :guilabel:`CMD` 的内容来判断，如果是 ``redis-server`` 的话，则切换到 ``redis`` 用户身份启动服务，否则依旧使用 root 身份执行。比如：

    .. code-block:: none

        $ docker container run --detach --publish-all --name kvstore redis:4.0-alpine
        1656bd2427fcc96e3e9dbdaf0e498786ca8817b5bb97e200476c555a117964c5

        $ docker container exec kvstore id
        uid=0(root) gid=0(root)
