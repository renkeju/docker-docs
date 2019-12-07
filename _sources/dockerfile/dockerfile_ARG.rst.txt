ARG 构建参数
^^^^^^^^^^^^^^^^^

格式： ``ARG <parameter>[=<defaults>]``

构建参数和 :guilabel:`ENV` 的效果一样，都是设置环境变量。所不同的是，:guilabel:`ARG` 所设置的构建环境的环境变量，在将来容器运行时是不会存在这些环境变量的。但是不要因此就使用 :guilabel:`ARG` 保存密码之类的信息，因为 ``docker image history IMAGE:TAG`` 还是可以看到所有值的。

:guilabel:`Dockerfile` 中的 :guilabel:`ARG` 指令是定义参数名称，以及定义其默认值。该默认值可以在构建命令 ``docker image build`` 中用 ``--build-arg <parameter>=<value>`` 来覆盖。

在 **1.13** 之前的版本，要求 ``--build-arg``中的参数名，必须在 :guilabel:`Dockerfile` 中用 :guilabel:`ARG` 定义过了，换句话说，就是 ``--build-arg`` 指定的参数，必须在 :guilabel:`Dockerfile` 中使用了。如果对应参数没有被使用，则会报错退出构建。从 **1.13** 开始，这种严格的限制被放开，不再报错退出，而是显示警告信息，并继续构建。这对于使用 CI 系统，用同样的构建流程构建不同的 :guilabel:`Dockerfile` 的时候比较有帮助，避免构建命令必须根据每个 Dockerfile 的内容修改。