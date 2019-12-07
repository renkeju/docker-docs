ADD 更高级的复制文件
^^^^^^^^^^^^^^^^^^^^^^^^

:guilabel:`ADD` 指令和 :guilabel:`COPY` 的格式和性质基本一致。但是在 :guilabel:`COPY` 基础上增加了一些功能。

比如 ``<source_path>`` 可以是一个 URL，这种情况下，Docker 引擎会试图去下载这个链接的文件放到 ``<destination_path>`` 去。下载后的文件权限自动设置为 **600**，如果这并不是想要的权限，那么还需要增加额外的一层 :guilabel:`RUN` 指令进行权限调整，另外，如果下载的是个压缩包，需要解压缩，也一样还需要额外的一层 :guilabel:`RUN` 指令进行解压缩。所以不如直接使用 :guilabel:`RUN` 指令，然后使用 wget 或者 curl 工具下载，处理权限、解压缩、然后清理无用文件更合理。因此，这个功能其实并不实用，而且不推荐使用。

如果 ``<source_path>`` 为一个 tar 压缩文件的话，压缩格式为 ``gzip`` ``bzip2`` 以及 ``xz`` 的情况下，:guilabel:`ADD` 指令将会自动解压缩这个压缩文件到 ``<destination_path>`` 去。

在某些情况下，这个自动解压缩的功能非常游泳，比如官方镜像 `ubuntu Dockerfile <https://github.com/tianon/docker-brew-ubuntu-core/blob/c7e9f7353aa24d1c35f501e06382aed1b540e85f/bionic/Dockerfile>`_ 中：

.. code-block:: none

    FROM scratch
    ADD ubuntu-bionic-core-cloudimg-amd64-root.tar.gz /
    ...

但在某些情况下，如果我们真的希望复制这个压缩文件进去，而不解压缩，这时就不可以使用 :guilabel:`ADD` 命令了。

在 Docker 官方的 Dockerfile 最佳实践文档 中要求，尽可能的使用 :guilabel:`COPY`，因为 :guilabel:`COPY` 的语义很明确，就是复制文件而已，而 :guilabel:`ADD` 则包含了更复杂的功能，其行为也不一定很清晰。最适合使用 :guilabel:`ADD` 的场合，就是所提及的需要自动解压缩的场合。

另外需要注意的是，:guilabel:`ADD` 指令会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢。

因此在 :guilabel:`COPY` 和 :guilabel:`ADD` 指令中选择的时候，可以遵循这样的原则，所有的文件复制均使用 :guilabel:`COPY` 指令，仅在需要自动解压缩的场合使用 :guilabel:`ADD`。