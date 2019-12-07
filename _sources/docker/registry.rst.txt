Docker Registry
~~~~~~~~~~~~~~~~~~~~~

Registry 分类
^^^^^^^^^^^^^^^^^^^

* Registry 用于保存 docker 镜像，包括镜像的层次结构和元数据。
* 用户可自建 Registry，也可以使用官方的 Docker Hub
* 详细分类

    * Sponsor Registry

        第三方的 Registry，供客户和 Docker 社区使用

    * Mirror Registry

        第三方的 Registry，只让客户使用

    * Vendor Registry

        由发布 Docker 镜像的供应商提供的 Registry

    * Private Registry

        通过设有防火墙和额外的安全层的私有实体提供的 Registry
    
私有仓库操作
^^^^^^^^^^^^^^^^^^^^^^^^

有时候使用 Docker Hub 这样的公共仓库可能不方便，用户可以创建一个本地仓库供私人仓库。

创建好私有仓库之后，就可以使用 ``docker tag`` 来标记一个镜像，然后推送它到仓库。例如私有仓库地址为 ```127.0.0.1:5000`` 。

先在本机查看已有的镜像。

.. code-block:: none

    $ docker image ls
    REPOSITORY     TAG      IMAGE ID            CREATED             VIRTUAL SIZE
    ubuntu         latest   ba5877dc9bec        6 weeks ago         192.7 MB

使用 ``docker image tag`` 将 ``ubuntu:latest`` 这个镜像标记为 ``127.0.0.1:5000/ubuntu:latest`` 。
格式为 ``docker image IMAGE[:TAG] [REGISTRY_HOST[:REGISTRY_PORT]/]REPOSITORY[:TAG]`` 。

.. code-block:: none

    $ docker tag ubuntu:latest 127.0.0.1:5000/ubuntu:latest
    $ docker image ls
    REPOSITORY                        TAG       IMAGE ID            CREATED             VIRTUAL SIZE
    ubuntu                            latest    ba5877dc9bec        6 weeks ago         192.7 MB
    127.0.0.1:5000/ubuntu:latest      latest    ba5877dc9bec        6 weeks ago         192.7 MB

使用 ``docker image push`` 上传标记的镜像

.. code-block:: none

    $ docker push 127.0.0.1:5000/ubuntu:latest
    The push refers to repository [127.0.0.1:5000/ubuntu]
    373a30c24545: Pushed
    a9148f5200b0: Pushed
    cdd3de0940ab: Pushed
    fc56279bbb33: Pushed
    b38367233d37: Pushed
    2aebd096e0e2: Pushed
    latest: digest: sha256:fe4277621f10b5026266932ddf760f5a756d2facd505a94d2da12f4f52f71f5a size: 1568

用 ``curl`` 查看仓库中的镜像。

.. code-block:: none

    $ curl 127.0.0.1:5000/v2/_catalog
    {"repositories":["ubuntu"]}

这里可以看到 ``{"repositories":"[ubuntu]"}`` ，这表明镜像已经被成功上传。

先删除已有镜像，在尝试从私有仓库中下载这个镜像。

.. code-block:: none

    $ docker image rm 127.0.0.1:5000/ubuntu:latest

    $ docker image pull 127.0.0.1:5000/ubuntu:latest
    Pulling repository 127.0.0.1:5000/ubuntu:latest
    ba5877dc9bec: Download complete
    511136ea3c5a: Download complete
    9bad880da3d2: Download complete
    25f11f5fb0cb: Download complete
    ebc34468f71d: Download complete
    2318d26665ef: Download complete

    $ docker image ls
    REPOSITORY                         TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
    127.0.0.1:5000/ubuntu:latest       latest              ba5877dc9bec        6 weeks ago         192.7 MB 

注意事项
^^^^^^^^^^^^^^^^

如果你不想使用 ``127.0.0.1:5000`` 作为仓库地址，比如想让本网段的其他主机也能把镜像推送到私有仓库。你就得把例如 ``192.168.199.100:5000`` 这样的内网地址作为私有仓库地址，这时你会发现无法成功推送镜像。

这是因为 Docker 默认 不允许非 HTTPS 方式推送镜像。我们可以通过 Docker 的配置选项来取消这个限制。

对于使用 ``systemd`` 的系统，请在 ``/etc/docker/daemon.json`` 中写入如下内容（如果文件不存在请新建该文件）

.. code-block:: none

    {
        "registry-mirror": [
            "https://registry.docker-cn.com"
        ],
        "insecure-registries": [
            "192.168.199.100:5000"
        ]
    }

.. note:: 

    该文件必须符合 json 规范，否则 Docker 将不能启动。