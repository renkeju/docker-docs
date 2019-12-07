ENV 设置环境变量
^^^^^^^^^^^^^^^^^^^^^^^

格式有两种：

* ``ENV <key> <value>``
* ``ENV <key1>=<value1> <key2>=<value2>...``

这个指令很简单，就是设置环境变量而已，无论是后面的其他指令，如 :guilabel:`RUN`，还是运行时的应用，都可以直接使用这里定义的环境变量。

.. code-block:: none

    ENV VERSION=1.0 DEBUG=on \
        NAME="Happy Feet"

这个例子中演示了如何换行，以及对含有空格的值用双引号扩起来的办法，这和 Shell 下的行为是一致的。

定义了环境变量，那么在后续的指令中，就可以使用这个环境变量。比如在官方 node 镜像 Dockerfile 中，就有类似这样的代码：

.. code-block:: none

    ENV NODE_VERSION 7.2.0

    RUN curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.tar.gz" \
        && curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/SHASUMS256.txt.asc" \
        && gpg --batch --decrypt --output SHASUMS256.txt SHASUMS256.txt.asc \
        && grep "node -v$NODE_VERSION-linux-x64.tar.xz\$" SHASUMS256.txt | sha256sum -c - \
        && tar -xJf "node-v$NODE_VERSION-linux-x64.tar.xz" -C /usr/local --strip-components=1 \
        && rm "node-v$NODE_VERSION-linux-x64.tar.xz" SHASUMS256.txt.asc SHASUMS256.txt \
        && ln -s /usr/local/bin/node /usr/local/bin/nodejs

在这里先定义了环境变量 ``NODE_VERSION`` ，其后的 :guilabel:`RUN` 这层里，多次使用 ``$NODE_VERSION`` 来进行操作定制。可以看到，将来升级镜像构建版本的时候，只需要更新 :guilabel:`7.2.0` 即可，:guilabel:`Dockerfile` 构建维护变得更轻松了。

下列指令可以支持环境变量的展开：

1. ADD
2. COPY
3. ENV
4. EXPOSE
5. LABEL
6. USER
7. WORKDIR
8. VOLUME
9. STOPSIGNAL
10. ONBUILD

可以从这个指令列表感觉到，环境变量可以使用的地方很多，很强大。通过环境变量，我们可以让一份 :guilabel:`Dockerfile` 制作更多的镜像，只需使用不同的环境变量即可。