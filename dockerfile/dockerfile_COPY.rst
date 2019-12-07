COPY 复制文件
^^^^^^^^^^^^^^^^^

格式：

* ``COPY <source_path>... <destination_path>``
* ``COPY ["<source_path_01>",... "<destination_path>"]``

和 :guilabel:`RUN` 指令一样，也有两种格式，一种类似于命令行，一种类似于函数调用。

:guilabel:`COPY` 指令将从构建上下文目录中 ``<source_path>`` 的文件/目录复制到新的一层的镜像内 ``<destination_path>`` 位置。比如：

.. code-block:: none

    COPY package.json /usr/src/app

``<destination_path>`` 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径（工作目录可以用 :guilabel:`WORKDIR` 指令来指定）。目标路径不需要事先创建，如果目录不存在会在复制文件前先进行创建缺失目录。

此外，还需要注意一点，使用 :guilabel:`COPY` 指令，源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。这个特性对于镜像定制很有用。特别是构建相关文件都在使用 Git 进行管理的时候。