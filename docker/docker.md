#### `docker`学习记录常用命令
> docker 启动停止
> systemctl stop docker
* docker 重启策略
  * --restart (参数)
    * no 容器退出时不重启
    * always 容器退出时总是自动重启
    * on-failure[:max-retry] 只在失败时重启
* 重启`docker`守护进程
  > docker daemon -H tcp://0.0.0.0:2375

* 使用socat 监控流量
  > sudo socat -v UNIX-LISTEN:/tcp/dockerapi.sock \
         UNIX-CONNECT:/var/run/docker.sock &
* 使用`docker` 查看请求与响应
  > docker -H unix:///tmp/dockerapi.sock ps -a
* docker 注册中心
  > docker run -d -p 5000:5000 -v $HOME/registry:var/lib/registry registry:2
  #修改 `--insecure-registry`允许使用`http`拉取
* 使用`BitTorrent Sync`的分布式卷
  * 配置第一台宿主机上的容器 开放必要的端口
  > [host1]  docker run -d -p 8888:8888 -p 55555:55555 \
            --name btsync ctlc/btsync
            docker logs btsync
            Starting btsync with secret: \
            ALSV********************** # 记下这个键后面用到
  * 启动一个交互式容器挂载`btsync`服务器的卷
  > [host1] docker run -i -t --volumes-from btsync \
          ubuntu /bin/bash
    * 添加一个文件到卷
    > touch /data/shared_from_server_one
    > ls /data
    > shared_from_server_one
   * 在次服务上开启一个终端同步卷
    >[host2] docker run -d --name btsync-client -p 8888:8888 \
    > -p 55555:55555 \
    > ctlc/btsync ALSV********************** # 上面的键
    > [host2] docker run -it --volumes-from btsync-client \
    > ubuntu bash
    > ls bash
    > shared_from_server_one
    > touch /data/share_from_server_two
    > ls /data
    > share_from_server_one shared_service_two

* 共享`bash`的历史
  * 宿主记得`bash`历史挂载在容器里
  > docker run -e HIST_FILE=/root/.bash_history \
  > -v=$HOME/.bash_history:/root/.bash_history \
  > -it ubuntu /bin/bash
* 挂载远程卷
* 需要`root`权限
  > docker run -ti --privileged debian /bin/bash
  > #在容器内安装`SSHFS`
  > apt-get update && apt-get install sshfs
  * 登录到远程主机
  > LOCALPATH=/path/to/local/directory
  > mkdir $LOCALPATH
  >
  > sshfs user@host:/path/to/remote/directory $LOCALPATH

* docker 构建 不需要缓存
  > docker build --no-cache .
* 将用户添加到组使用`docker`命令
  > sudo addgroup -a username docker

* docker 删除挂载关联卷
  > docker rm -v

* docker 运行可视化界面`DockerUI`
  > docker run -d -p 9000:9000 --privileged \
  > -d /var/run/docker.sock:/var/run/docker.sock dockerui/dockerui

* docker 生成镜像分层树
  > docker run --rm \
  > -v /var/run/docker.sock:/var/run/docker.sock \
  > dockerinpractice/docker-image-graph > docker_images.png
* docker 删除一周前的日志
  > docker exec -d sleeper \
  > find / -ctime 7 -name '*log' -exec rm {} \; #删除最近7天没有做过更改并且以log结尾的文件

> docker history mysecret # 查看容器的创建的历史
* docker 导出容器 并导入 进行脱密处理
> docker export 28cde380f | docker import - mysecret
* 需要保留`docker`要使用`load` 不应该使用`import`
* docker 限制使用资源cpu
  > docker run --cpuset-cpus=0 -c 10000 ubuntu:14.04 \
  > sh -c 'cat /dev/zero > /dev/null' & 
  > docker run --cpuset-cpus=0 -c 1 -it ubuntu:14.04 bash

* docker 显示内存使用
  > docker run-it -m 100m ubuntu:14.04 bash

* 使用`cron` 定时执行作业
  > 0 0 * * * \
  > docker pull docker dockerinpractice/log_cleaner && \
  > docker run \
  > -v /var/log/myapplogs:/log_dir dockerinpractice/log_cleaner 1
* 提交和推送备份容器
  > 产生一个记录到秒的时间戳
  > DATE=$(date +%Y%m%d_%H%M%S)
  > 通过代用宿主机名和日期的标签产生一个指向你的注册中心URL的标签
  > TAG="your_log_registry:5000/live_pmt_svr_backup:$(hostname -s)_${DATE}"
  >
  > 以日期为消息，以Backup Admin 为作者，提交容器
  > docker commit -m="${DATE}" -a="Backup Admin" live_pmt_svr $TAG
  > docker push $TAG #推送注册中心