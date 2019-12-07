HEALTHCHECK 健康检查
^^^^^^^^^^^^^^^^^^^^^^^^^^^

格式：

* ``HEALTHCHECK [option] CMD <COMMAND>`` 设置检查容器监控状态的命令
* ``HEALTHCHECK NONE`` 如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令

:guilabel:`HEALTHCHECK` 指令是告诉 Docker 应该如何进行判断容器的状态是否正常，这是从 **Docker 1.12** 引入的新指令。

在没有 :guilabel:`HEALTHCHECK` 指令前，Docker 引擎只可以通过容器内主进程是否退出来判断容器是否状态异常。很多情况下没有问题，但是如果程序进入了死锁状态，或者死循环状态，应用进程并不退出，但是该容器已经无法提供服务了。在 **1.12** 以前，Docker 不会检测到容器的这种状态，从而不会重新调度，导致可能会有部分容器已经无法提供服务了却还在接受用户请求。

而自 **1.12** 之后，Docker 提供了 :guilabel:`HEALTHCHECK` 指令后，用其启动容器，初始状态会为 :guilabel:`starting`，在 :guilabel:`HEALTHCHECK` 指令检查成功后变为 :guilabel:`healthy` ，如果连续一定次数失败，则会变为 :guilabel:`unhealthy`。

:guilabel:`HEALTHCHECK` 支持下列选项：

* ``--interval=<second>`` 两次健康检查的间隔，默认为 30 秒；
* ``--timeout=<second>`` 健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
* ``--retries=<number>`` 当连续失败指定次数后，则将容器状态视为 :guilabel:`unhealthy`，默认 3 次。

和 :guilabel:`CMD` :guilabel:`ENTRYPOINT` 一样，:guilabel:`HEALTHCHECK` 只可以出现一次，如果写了多个，只有最后一个生效。

在 ``HEALTHCHECK [option] CMD`` 后面的命令，格式和 :guilabel:`ENTRYPOINT` 一样，分为 **shell** 格式，和 **exec** 格式。命令的返回值决定了该次健康检查的成功与否

==== ===========
code  status
==== ===========
0     success
1     fail
2     save
==== ===========

.. attention:: 

    不要使用 2 这个值

假设我们有个镜像是个最简单的 Web 服务，我们希望增加健康检查来判断其 Web 服务是否在正常工作，我们可以用 curl 来帮助判断，其 :guilabel:`Dockerfile` 的 :guilabel:`HEALTHCHECK` 可以这么写：

.. code-block:: none

    FROM nginx:1.14.0

    RUN apt-get update \
        && apt-get install -y curl \
        && rm -rf /var/lib/apt/lists/*

    HEALTHCHECK --interval=5s --timeout=3s \
            CMD curl -fs http://localhost/ || exit 1

这里我们设置了每5秒检查一次（这里为了试验所以间隔非常短，实际应该相对较长），如果健康检查命令超过3秒没有响应就视为失败，并且使用 ``curl -fs http://localhost/ || exit 1`` 作为健康检查命令。

使用 ``docker image build`` 来构建这个镜像：

.. code-block:: none

    docker image build --tag=myweb:1.0 .

构建完成之后，我们启动一个容器：

.. code-block:: none

    container run --detach --publish 80:80 --name myweb_test myweb:1.0

当运行该镜像后，可以通过 ``docker container ls`` 看到最初的状态为 ``(health: starting)`` ：

.. code-block:: none

    docker container ls
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                   PORTS                     NAMES
    6d9d2d0557de        myweb:1.0           "nginx -g 'daemon of…"   10 seconds ago      Up 9 seconds (healthy)   0.0.0.0:80->80/tcp        myweb_test

在等待几秒钟后，再次 ``docker container ls`` 就会看到健康状态变化为了 ``(healthy)`` ：

.. code-block:: none

    docker container ls
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                    PORTS                     NAMES
    6d9d2d0557de        myweb:1.0           "nginx -g 'daemon of…"   22 seconds ago      Up 21 seconds (healthy)   0.0.0.0:80->80/tcp        myweb_test

如果健康检查连续失败超过了重试次数，状态就会变为 ``(unhealthy)`` 。

为了帮助排障，健康检查命令的输出（包括 stdout 以及 stderr） 都会被存储在健康状态里，可以用 ``docker container inspect CONTAINER`` 来查看。

.. code-block:: none

    docker container inspect --format '{{json .State.Health}}' myweb_test | python -m json.tool
    {
        "Status": "healthy",
        "FailingStreak": 0,
        "Log": [
            {
                "Start": "2018-11-01T09:13:48.06308Z",
                "End": "2018-11-01T09:13:48.1525581Z",
                "ExitCode": 0,
                "Output": "<!DOCTYPE html>\n<html>\n<head>\n<title>Welcome to nginx!</title>\n<style>\n    body {\n        width: 35em;\n        margin: 0 auto;\n        font-family: Tahoma, Verdana, Arial, sans-serif;\n    }\n</style>\n</head>\n<body>\n<h1>Welcome to nginx!</h1>\n<p>If you see this page, the nginx web server is successfully installed and\nworking. Further configuration is required.</p>\n\n<p>For online documentation and support please refer to\n<a href=\"http://nginx.org/\">nginx.org</a>.<br/>\nCommercial support is available at\n<a href=\"http://nginx.com/\">nginx.com</a>.</p>\n\n<p><em>Thank you for using nginx.</em></p>\n</body>\n</html>\n"
            },
            ...
        ]
    }