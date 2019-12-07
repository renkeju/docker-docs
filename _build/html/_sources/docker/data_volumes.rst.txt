存储卷
~~~~~~~~~~~~~~

* Docker 镜像由多个只读层叠加而成，启动容器时，Docker 会加载只读镜像层并在镜像栈顶部添加一个读写层
* 如果运行中的容器修改了现有的一个已存在的文件，那该文件将会从读写层下面的只读层复制到读写层，该文件的只读版本仍然存在，只是已经被读写层中该文件的副本所隐藏，此即“写时复制（COW）”机制。

.. image:: /images/container/docker/Copy_on_write.png

COW 这种机制我们增删改查等一类的操作必然会降低文件系统的效率，那带来的是那些对I/O要求较高的应用（例如：Redis\MySQL）实现持久化存储的时候，这样的存储系统性能会大打折扣。

如果要绕过这种机制，我们可以通过存储卷来实现。特全级的名称空间当中，我们也可以理解为宿主机当中找一个本地文件系统，创建一个本地目录，把这个目录与容器内部的文件系统的某一目录创建绑定关系。

什么是卷
^^^^^^^^^^^^^^^^^^^^^^^

* 关闭并重启容器，其数据不受影响；但删除 Docker 容器，则其更改将会全部丢失
* 存在的问题

    * 存储与联合文件系统中，不易被宿主机访问
    * 容器间数据共享不便
    * 删除容器其数据会丢失

* 解决方案 “卷”

    “卷” 是容器上的一个或多个“目录”，此类目录可以绕过联合文件系统，与宿主机上的某目录“绑定（关联）”

有状态服务 VS 无状态服务
^^^^^^^^^^^^^^^^^^^^^^^^^^^

* 无状态服务（Stateless Service）

    是指该服务运行的实例不会在本地存储需要持久化的数据，并且多个实例对于同一个请求响应的结果是完全一致的。这类服务在 k8s 平台创建后，借助 k8s 内部的负载均衡，当访问该服务的请求到达服务一端后，负载均衡会随机找到一个实例来完整该请求的响应（目前为轮询）。这类服务的实例可能会应为一些原因停止或者重新创建（如扩容时），这时，这些停止的实例里的所有信息（除日志和监控数据外）都将丢失（重启容器就会丢失）。因此如果您的容器实例里需要保留重要的信息，并希望随时可以备份以便与以后可以恢复的话，那么建议您创建有状态服务。

* 有状态服务（Stateful Service）

    是指该服务的实例可以将一部分数据随时进行备份，并且在创建一个新的有状态服务时，可以通过备份恢复这些数据，以达到数据持久化的目的。有状态服务只能有一个实例，因此不支持“自动服务容量调节”。一般来说，数据库服务或者需要在本地文件系统存储配置文件或其他永久数据的应用程序可以创建使用有状态服务。想要创建有状态服务，必须满足几个前提：

    待创建的服务镜像（image）的 Dockerfile 中必须定义了存储卷（Volume），因为只有存储所在目录里的数据可以被备份。
    创建服务时，必须指定给该存储卷分配的磁盘空间大小。
    如果创建服务的同时需要从之前的一个备份里恢复数据，那么还要指明该存储卷用哪个备份恢复。

* 无状态服务和有状态服务主要有以下几点区别

    1. 实例数量

        无状态服务可以有一个或多个实例，因此支持两种服务容量调节模式；有状态服务智能有一个实例，不允许创建多个实例，因此也不支持服务容量调节模式。

    2. 存储卷

        无状态服务可以有存储卷，也可以没有，即使有也无法备份存储卷里面的数据；有状态服务必须有存储卷，并且在创建服务时，必须指定给该存储卷分配的磁盘的空间大小。

    3. 数据存储

        无状态服务在运行过程中的所有数据（除日志和监控数据）都存在容器实例的文件系统中，如果实例停止或删除，则这些数据都将丢失，无法找回；而对于有状态服务，凡是已经挂载了存储卷的目录下的文件内容都可以随时进行备份，备份的数据可以下载，也可以用于恢复新的服务。但对于没有挂载卷的目录下的数据，仍然时无法备份和保存的，如果实例停止或者删除，这些非挂载卷里的文件内容同样会丢失。

volume 的几种形态
^^^^^^^^^^^^^^^^^^^^^

有状态容器有数据持久化需求。Docker 采用 AFUS 分层文件系统时，文件系统的改动都是发生在最上面的容器层。在容器的声明周期内，他是持续的，包括容器在被停止后。但是，当容器被删除后，该数据层也随之被删除了。因此，Docker 采用 volume（卷）的形式来向容器提供持久化存储。Docker volume 有如下几种状态。

1. 无 —— 不使用 Docker volume

    默认情况下，容器不使用任何 volume，此时，容器的数据被保存在容器之内，它只在容器的生命周期内存在，会随着容器的被删除而删除。当然，也可以使用 docker commit 命令将它持久化为一个新的镜像。

2. Data volume（数据卷）

    一个 data volume 是容器中绕过 Union 文件系统的一个特定的目录。它被设计用来保存数据，而不管容器的生命周期。因此，当你删除一个容器时，Docker 肯定不会自动地删除一个 volume。有如下几种方式来使用 data volume：

    * 使用 ``-v local_file:container_file`` 形式

        .. code-block:: bash 

            docker container run --detach --publish-all --name kvstor --volume ~/Documents/redis/redis.conf:/usr/local/etc/redis/redis.conf redis:4.0-alpine

        .. code-block:: none

            docker container inspect kvstor 
            ...
            "Mounts": [
                {
                    "Type": "bind",
                    "Source": "/Users/renkeju/Documents/docker-compose/redis/redis.conf",
                    "Destination": "/usr/local/etc/redis/redis.conf",
                    "Mode": "",
                    "RW": true,
                    "Propagation": "rprivate"
                },
            ...

    * 使用 ``-v container_dir`` 形式 Docker-managed volume

        .. code-block:: bash 

            docker container run --detach --name web01 --volume /webapp nginx

        .. code-block:: none

            "Mounts": [
                {
                    "Type": "volume",
                    "Name": "fb00ff0ceb59cc1e1f1cb995ccddf071660146142f64b2a6e81037b37454c614",
                    "Source": "/var/lib/docker/volumes/fb00ff0ceb59cc1e1f1cb995ccddf071660146142f64b2a6e81037b37454c614/_data",
                    "Destination": "/webapp",
                    "Driver": "local",
                    "Mode": "",
                    "RW": true,
                    "Propagation": ""
                }
            ],
    
        其实，在 web01 容器被删除后，/var/lib/docker/volumes/fb00ff0ceb59cc1e1f1cb995ccddf071660146142f64b2a6e81037b37454c614/_data 目录及其中的内容都还会保留下来，但是新启动的容器无法再使用这个目录，也就是说，已有的数据不能自动地被重复使用。

    * 使用 ``-v local_dir:container_dir`` 形式 Bind mount volume

        .. code-block:: none

            docker container run --publish-all --detach --name web02 --volume /Users/renkeju/Documents/HarborCloud-docs/build/html:/usr/share/nginx/html nginx

        .. code-block:: none

            docker container inspect web02
            ...
            "Mounts": [
                {
                    "Type": "bind",
                    "Source": "/Users/renkeju/Documents/HarborCloud-docs/build/html",
                    "Destination": "/usr/share/nginx/html",
                    "Mode": "",
                    "RW": true,
                    "Propagation": "rprivate"
                }
            ],
            ...

        主机上的目录可以时一个本地目录，也可以在一个 NFS Share 内，或者在一个已经格式化好了的块设备上。

    其实这种形式和第一种没有本质的区别，容器内对 /usr/share/nginx/html 的操作都会反应到主机的 ~/Documents/HarborCloud-docs/build/html 目录内。只是，重新启动容器时，可以再次使用同样的方式来将 ~/Documents/HarborCloud-docs/build/html 目录挂载到新的容器内，这样就可以实现数据持久化的目标。

3. 使用 data container 

    如果要在容器之间共享数据，最好是使用 data container。这种 container 中不会跑应用，而只是挂载一个卷。比如：

    创建一个 data container

    .. code-block:: bash 

        docker container create --volume /dbdata --name dbstore busybox 

    启动一个 app container

    .. code-block:: bash

        docker container run --detach --publish-all --name web03 --volumes-from dbstore nginx

    其实，对 web03 这个容器来说，volume 的本质没有变，它只是将 dbstore 容器的 /dbdata 目录映射的主机上的目录映射到自身的 /dbdata 目录。

    .. code-block:: none

        "Mounts": [
            {
                "Type": "volume",
                "Name": "47373e7814371d703fe7a94b2282eecb3dbce122ae1faf001739231f054b8d42",
                "Source": "/var/lib/docker/volumes/47373e7814371d703fe7a94b2282eecb3dbce122ae1faf001739231f054b8d42/_data",
                "Destination": "/dbdata",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],

    这样做的优势是不管其目录的临时性而不断地重复使用它。

4. 使用 docker volume 命令

    Docker 新版本中引入了 docker volume 命令来管理 Docker volume

    * 使用默认的 ``local`` driver 创建一个 volume

        .. code-block:: none

            docker volume create --name vol1
            docker volume inspect vol1
            [
                {
                    "CreatedAt": "2018-10-30T05:05:54Z",
                    "Driver": "local",
                    "Labels": {},
                    "Mountpoint": "/var/lib/docker/volumes/vol1/_data",
                    "Name": "vol1",
                    "Options": {},
                    "Scope": "local"
                }
            ]

    * 使用这个 volume

        .. code-block:: none

            docker container run --detach --publish-all --name web04 --volume vol1:/volume nginx

        结果还是相同，将 vol1 对应的主机上的目录挂载给容器内的 /volume 目录

        .. code-block:: none

            "Mounts": [
                {
                    "Type": "volume",
                    "Name": "vol1",
                    "Source": "/var/lib/docker/volumes/vol1/_data",
                    "Destination": "/volume",
                    "Driver": "local",
                    "Mode": "z",
                    "RW": true,
                    "Propagation": ""
                }
            ],

volume 删除和孤独 volume 清理
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1. 在删除容器的时候 volume

    可以使用 ``docker container rm -v volume_name`` 命令在删除容器的时候删除该容器的卷

    .. code-block:: bash 

        docker container --volumes --force web04

    .. code-block:: bash

        docker volume ls
        DRIVER              VOLUME NAME
        local               fb00ff0ceb59cc1e1f1cb995ccddf071660146142f64b2a6e81037b37454c614
        local               vol1

2. 批量删除孤独的 volumes

    从上面的介绍可以看出，使用 ``docker container run --volume`` 启动的容器被删除以后，在主机上会遗留下来孤单的卷。可以使用下面的简单方法来做清理：

    .. code-block:: bash 

        docker volume ls --quiet --filter dangling=true
        01e516feecef3701b84e766a21dd2988c53978e223070fed2636eaba56109c5e
        b08f68c5170e417ce4c4aab6667c1aadebb4a2cf6af39099d7b7e3dc36c6b74a
        fb00ff0ceb59cc1e1f1cb995ccddf071660146142f64b2a6e81037b37454c614
        vol1
        docker volume rm $(docker volume ls --quiet --filter dangling=true)
        01e516feecef3701b84e766a21dd2988c53978e223070fed2636eaba56109c5e
        b08f68c5170e417ce4c4aab6667c1aadebb4a2cf6af39099d7b7e3dc36c6b74a
        fb00ff0ceb59cc1e1f1cb995ccddf071660146142f64b2a6e81037b37454c614
        vol1
        docker volume ls
        DRIVER              VOLUME NAME
        local               47373e7814371d703fe7a94b2282eecb3dbce122ae1faf001739231f054b8d42

    也可以直接使用自带的清楚策略

    .. code-block:: bash 

        docker volume prune
        WARNING! This will remove all local volumes not used by at least one container.
        Are you sure you want to continue? [y/N] y
        Deleted Volumes:
        vol1

        Total reclaimed space: 0B    