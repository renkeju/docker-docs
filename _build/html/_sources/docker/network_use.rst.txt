网络基础使用
~~~~~~~~~~~~~~

与 bridge 相关的参数
^^^^^^^^^^^^^^^^^^^^^^

可以为 ``docker container run`` 命令使用

* ``--hostname HOSTNAME`` 选项为容器指定主机名，例如

    .. code-block:: bash

        docker container --rm --net=bridge --hostname=bbox.renkeju.com busybox:latest nslookup bbox.renkeju.com

* ``--dns DNS_SERVER_IP`` 选项能够为容器指定所使用的 dns 服务器地址，例如

    .. code-block:: bash

        docker container run --dns 8.8.8.8 busybox:latest nslookup docker.com

* ``--add-host HOSTNAME:IP`` 选项能够为容器指定本地主机名解析项，例如

    .. code-block:: bash

        docker container run --rm --dns 8.8.8.8 -add-host "docker.com:172.16.0.100" busybox:latest nslookup docker.com

打开入站通讯
^^^^^^^^^^^^^^^^^^^^

* ``--publish`` 选项使用格式

    * ``--publish <ContainerPort>``

        将制定的容器端口映射至主机所有地址的一个动态端口

    * ``--publish <HostPort>:<ContainerPort>``

        将容器端口 <ContainerPort> 映射至指定的主机端口 <hostPort>

    * ``--publish <ip>::<ContainerPort>``

        将指定的容器端口 <ContainerPort> 映射至主机指定 <ip> 的动态端口

    * ``--publish <ip>:<hostPort>:<containerPort>``

        将指定的容器端口 <containerPort> 映射至主机指定 <ip> 的端口 <hostPort>

“动态端口” 指的是随机端口，具体的映射结果可以使用 docker port 命令查看

.. code-block:: bash

    $ docker container port wordpresss_wordpress_1
    80/tcp -> 0.0.0.0:80

* ``--publish-all`` 选项将容器的所有计划要暴露端口全部都映射至主机端口
* 计划要暴露的端口使用 ``--expose`` 选项指定

    .. code-block:: bash

        docker container run --detach --publish-all --expose 3333 --expose 2222 --name=web01 busybox:latest /bin/httpd -p 2222 -f

    * 查看映射结果

        .. code-block:: bash

            docker container port web01
            2222/tcp -> 0.0.0.0:32772
            3333/tcp -> 0.0.0.0:32771

* 如果不想使用默认的 docker0 桥接口，或者需要修改此桥接口的网络属性，可以通过 ``docker daemon`` 命令使用 ``-b`` ``--bip`` ``--fixed-cidr`` ``--default-gateway`` ``--dns`` ``--mtu`` 等选项进行设定。也可以通过修改 ``/etc/docker/daemon.json`` 配置文件设定。
* docker 守护进程的 C/S，默认仅监听 Unix Socket 格式的地址 ``/var/run/docker.sock`` 

    如果使用 TCP 套接字，需要修改 ``/etc/docker/daemon.json`` 文件。
    使用 ``-H|--host`` 选项可远程链接开启 TCP 套接字文件的 docker server。

