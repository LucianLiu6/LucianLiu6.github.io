---
title: setup-jupyter-in-docker
date: 2019-07-10 14:20:14
tags:
---

# Setup Jupyter in Docker

## 获取 Jupyter 镜像并启动
如果在 [Docker Hub](https://hub.docker.com/r/jupyter) 中搜索 `jupyter` 会有很多官方镜像，我们从中选一个名为 `jupyter/minimal-notebook` 的镜像来构建容器
<!-- more -->
创建容器的命令如下：

```shell
docker run -d -p ${host_exposed_port}:8888 --name=${container_name} -v ${host_volumn}:/home/jovyan jupyter/minimal-notebook
```
简单说下启动命令里面的几个参数
* `host_exposed_port` - 可选  
    需要暴露的宿主机端口，供其它机器访问本 Jupyter notebook  
    不需要的话就去掉 `-p ${host_exposed_port}:8888` 这个参数
* `container_name` - 可选
    容器名，方便在容器列表中查看/查找本容器  
    不需要的话就去掉 `--name=${container_name}` 这个参数
* `host_volumn` - 可选  
    将宿主机的目录挂载到容器中的 `/home/jovyan` 目录，这样在容器里面创建文件就等于在 `${host_volumn}` 创建，方便对 Notebook 的内容进行导入导出。

    这里要注意 `${host_volumn}` 的权限问题，需要保证容器启动用户有该目录的读写权限，不然创建容器的时候会失败，启动之后使用 `docker ps` 查看正在运行的容器会找不到刚创建的容器。  
    使用 `docker ps -a` 找到容器 ID 后再使用 `docker logs ${CONTAINER-ID}` 查看容器的 Log 可以看到错误信息 `PermissionError: [Errno 13] Permission denied: '/home/jovyan/.local'`，原因就是容器启动用户没有该目录的读写权限，解决办法就是给容器启动用户加上该目录的读写权限。添加权限的方法有很多，这里简单介绍一个
    ```shell
    cd ${host_volumn} && chmod o+rw .
    ```

    不需要的话就去掉 `-v ${host_volumn}:/home/jovyan` 这个参数，**强烈建议**配置这个参数方便 notebook 数据的导入导出

## 登录 Jupyter Notebook
安装好之后就可以通过 `http://${host_name}:${host_exposed_port}` 来访问 Jupyter Notebook 了，第一次访问需要提供 Login Token，Token 会在 Notebook 启动的时候输出到控制台，查看容器的日志即可找到 (`docker logs ${CONTAINER-ID}`)。正常可以看到这样的页面：

![set-password](set-password.png)

通过 Token 登录很不方便，如果下次换了个设备登录又要重新输入 Token，所以我们需要自己设个容易记忆的密码；如果你有一个可以记住 48 位 Token 的最强大脑的话就可以不用往下看了。像我这种凡夫俗子还是规规矩矩用密码吧

页面底部不就有个 `Setup a Password` 嘛，还用你在这儿叨叨？  
最开始我也是这么想的，把 Token 和 Password 往框框里面一填，开心地点了下 **Log in and set new password** ，结果得到了一个无情的 `500: Internal Server Error`

![unexpected-500](unexpected-500.png)
**??? What the hell ???**
 
淡定淡定，先看看容器的 Log (`docker logs ${CONTAINER-ID}`)，果然发现了猫腻所在
```shell
FileNotFoundError: [Errno 2] No such file or directory: '/home/jovyan/.jupyter/jupyter_notebook_config.json'
```
哦，原来是没有这个配置文件，所以密码写不进去，那我们就手动给它创建下呗。  
创建的方法其实有 2 种：
* 在 Docker 容器中创建  
    * 进入容器内部  
        `docker exec -it ${CONTAINER-ID} /bin/sh`
    * 创建 `.jupyter` 目录  
        `mkdir -p /home/jovyan/.jupyter`
* 在宿主机中创建  
    我们注意看下这个目录 `/home/jovyan`，这不就是我们前面创建容器时挂载到容器的宿主机目录吗？那我直接在宿主机创建目录不是更快？一句命令搞定  
    `mkdir -p ${host_volumn}/.jupyter`

如果第一次已经使用了 Token 登录，后面想再设置密码却找不到登录的那个页面了，怎么设置密码呢？
* 找回登录页  
    用浏览器开个无痕窗口访问页面即可
* 直接改配置文件 - 同样适用于修改密码的场景  
    看看官方文档怎么说 - [how-to-enable-password](https://jupyter-notebook.readthedocs.io/en/stable/public_server.html)

## 一些额外的需求

### 如果要在容器中安装一些自己需要的软件包怎么办？
Start minimal-notebook with **sudo access**

Add the param when start container:
```shell
-e GRANT_SUDO=yes --user root
```
[Reference](https://github.com/kubeflow/kubeflow/issues/425)
