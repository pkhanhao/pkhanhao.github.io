---
layout: post
title:  "体验LXD容器"
date:   2016-06-27 20:02:00 +0800
categories: jekyll update
---

Ubuntu 对[ LXD ][LXD-introduction]支持友好，这里使用 Ubuntu 16.04 LTS。了解 LXD 和 docker 容器的异同请看[这里][experience-lxd-in-ustack]，你也可以[在线体验 LXD ][try-lxd]。

### 1、获取 lxd apt 源

    $ add-apt-repository ppa:ubuntu-lxc/lxd-git-master
    $ cat ubuntu-lxc-lxd-git-master-trusty.list
    deb http://ppa.launchpad.net/ubuntu-lxc/lxd-git-master/ubuntu xenial main
    # deb-src http://ppa.launchpad.net/ubuntu-lxc/lxd-git-master/ubuntu xenial main

### 2、下载 lxd 相关包

> 如果想让普通用户可直接执行 lxc 命令，需将用户加入 lxd 组。

    $ aptitude install lxd lxd-client -y

### 3、lxd 网络配置

早期 lxd 网络依赖 lxc1 这个包，lxc1 分配一个 lxcbr0 桥接口。新版本 lxd 去掉了对 lxc1 的依赖，分配一个 lxdbr0 桥接口，并且只在 lxd 启动时激活，配置文件为 /etc/default/lxd-bridge 。当然，你也可以自定义桥接，并连接一个以太网口。

    $ dpkg-reconfigure -p medium lxd

### 4、lxd 镜像

查看镜像源。

    $ lxc remote list
    +-----------------+------------------------------------------+---------------+--------+--------+
    |      NAME       |                   URL                    |   PROTOCOL    | PUBLIC | STATIC |
    +-----------------+------------------------------------------+---------------+--------+--------+
    | images          | https://images.linuxcontainers.org       | lxd           | YES    | NO     |
    +-----------------+------------------------------------------+---------------+--------+--------+
    | local (default) | unix://                                  | lxd           | NO     | YES    |
    +-----------------+------------------------------------------+---------------+--------+--------+
    | ubuntu          | https://cloud-images.ubuntu.com/releases | simplestreams | YES    | YES    |
    +-----------------+------------------------------------------+---------------+--------+--------+
    | ubuntu-daily    | https://cloud-images.ubuntu.com/daily    | simplestreams | YES    | YES    |
    +-----------------+------------------------------------------+---------------+--------+--------+

查看远端镜像源 images 中的镜像。

    $ lxc image list images:

获取镜像源 images 中 debian/jessie/amd64 ，本地命名为 debian 。

    $ lxc image copy images:debian/jessie/amd64 local: --alias debian [--auto-update]

查看镜像源 local 中的镜像。

    $ lxc image list
    +--------+--------------+--------+---------------------------------------------+--------+----------+-------------------------------+
    | ALIAS  | FINGERPRINT  | PUBLIC |                 DESCRIPTION                 |  ARCH  |   SIZE   |          UPLOAD DATE          |
    +--------+--------------+--------+---------------------------------------------+--------+----------+-------------------------------+
    | debian | e97312cc09f7 | no     | Debian jessie (amd64) (20160614_22:42)      | x86_64 | 104.05MB | Jun 15, 2016 at 11:10am (UTC) |
    +--------+--------------+--------+---------------------------------------------+--------+----------+-------------------------------+
    |        | c1c8e9dc7ee5 | no     | Centos 6 (amd64) (20160615_02:16)           | x86_64 | 65.73MB  | Jun 16, 2016 at 1:42am (UTC)  |
    +--------+--------------+--------+---------------------------------------------+--------+----------+-------------------------------+

对 c1c8e9dc7ee5 命名（alias）为 centos6 。

    $ lxc image alias create centos6 c1c8e9dc7ee5
    $ lxc image list
    +---------+--------------+--------+---------------------------------------------+--------+----------+-------------------------------+
    |  ALIAS  | FINGERPRINT  | PUBLIC |                 DESCRIPTION                 |  ARCH  |   SIZE   |          UPLOAD DATE          |
    +---------+--------------+--------+---------------------------------------------+--------+----------+-------------------------------+
    | debian  | e97312cc09f7 | no     | Debian jessie (amd64) (20160614_22:42)      | x86_64 | 104.05MB | Jun 15, 2016 at 11:10am (UTC) |
    +---------+--------------+--------+---------------------------------------------+--------+----------+-------------------------------+
    | centos6 | c1c8e9dc7ee5 | no     | Centos 6 (amd64) (20160615_02:16)           | x86_64 | 65.73MB  | Jun 16, 2016 at 1:42am (UTC)  |
    +---------+--------------+--------+---------------------------------------------+--------+----------+-------------------------------+

从镜像导出 tar 包格式，metadata 是较小的那个。

    $ lxc image export centos6/c1c8e9dc7ee5

从 tar 包导入镜像，2个 tar 包，metadata 是较小的那个。

    $ lxc image import <metadata tarball> <rootfs tarball> --alias image-alias

### 5、lxd 容器

从远端源 images 启动容器，对应的镜像会在 local 中缓存一份。

    $ lxc launch images:centos/6/amd64 my-centos6

镜像源 local 启动容器，你也可以用 lxc init debian my-debian8 初始化一个容器但并不启动它。

    $ lxc launch debian my-debian8

查看本地容器。

    $ lxc list
    +------------+---------+---------------------+------+------------+-----------+
    |    NAME    |  STATE  |        IPV4         | IPV6 |    TYPE    | SNAPSHOTS |
    +------------+---------+---------------------+------+------------+-----------+
    | my-debian8 | RUNNING | 10.8.150.46 (eth0)  |      | PERSISTENT | 0         |
    +------------+---------+---------------------+------+------------+-----------+
    | my-cemtos6 | RUNNING | 10.8.150.122 (eth0) |      | PERSISTENT | 0         |
    +------------+---------+---------------------+------+------------+-----------+

查看容器信息。

    $ lxc info my-centos6

容器执行命令，这里获取了一个 bash 环境。

    $ lxc exec my-centos6 -- /bin/bash

将容器中 /etc/hosts 文件获取到本地。

    $ lxc file pull my-centos6/etc/hosts .

将当前 test 文件上传到容器 /tmp/ 下。

    $ lxc file push test my-centos6/tmp/

直接编辑容器中 /tmp/test 文件。

    $ lxc file edit my-centos6/tmp/test
    $ lxc exec my-centos6 -- ls /tmp/test
    /tmp/test

容器生命周期管理。

    $ lxc start|stop|restart|pause|delete <container>

复制容器 c1 ，新容器名为 c2 。

    $ lxc copy c1 c2

重命名 c2 容器为 c3 ，需 stop 容器先。

    $ lxc move c2 c3

从容器生成镜像，也可以从容器快照生成镜像。

    $ lxc publish my-container --alias my-new-image

### 6、lxd 快照

对容器 c1 做快照 c1-s1 。

    $ lxc snapshot c1 c1-s1

查看容器资源使用，可查看快照信息。

    $ lxc info c1

对容器做一些破环操作。

    $ lxc exec c1 -- rm -Rf /etc /usr

用快照 c1-s1 恢复容器 c1 。

    $ lxc restore c1 c1-s1

重命名快照。

    $ lxc move c1/c1-s1 c1/c1-1

从快照 c1/c1-1 生成新容器名为 c3 。

    $ lxc copy c1/c1-1 c3

### 7、lxd 资源配置

lxd 容器可配置资源、磁盘、开机启动等[设置项][LXD-configuration]，除非官方文档明确申明，通常这些配置都可以热更新，不用重启容器。

查看可用的 profile 配置，profile 配置一般设置通用设置项。

    $ lxc profile list
    default
    docker

查看 default 配置的设置项内容。

    $ lxc profile show default
    name: default
    config: {}
    description: Default LXD profile
    devices:
      eth0:
        name: eth0
        nictype: bridged
        parent: lxdbr0
        type: nic

配置 profile 设置项。

    $ lxc profile set <container> <key> <value>

编辑 profile 配置。

    $ lxc profile edit <profile>

应用 profile 配置到容器。

    $ lxc profile apply <container> <profile1>,<profile2>,...

设置 config 配置限制容器只可使用2个 cpu，config 配置一般设置容器特有设置项。

    $ lxc config set <container> limits.cpu 2

查看容器配置。

    $ lxc config show <container>
    name: centos
    profiles:
    - default
    config:
      limits.cpu: "2"
      volatile.base_image: 5214b823f2ef7a11ae34abf9a878f8d3d53f6badc1a4dfe76715baaa67766723
      volatile.eth0.hwaddr: 00:16:3e:9c:48:8c
      volatile.last_state.idmap: '[{"Isuid":true,"Isgid":false,"Hostid":1345184,"Nsid":0,"Maprange":65536},{"Isuid":false,"Isgid":true,"Hostid":1345184,"Nsid":0,"Maprange":65536}]'
    devices:
      root:
        path: /
        type: disk
    ephemeral: false

编辑 config 配置。

    $ lxc config edit <container>

增加 device ，具体语法可查看 lxc help config 。

    $ lxc config device add <container> ...

### 8、在线迁移

LXD 可以跨主机[在线迁移][LXD-live-migration]，不过这个特性目前还为试验性质，可能会无法正常工作。如果想要了解更多关于 LXD 的使用、特性信息，可以查看[ LXD 2.0: Blog post series ][LXD-2-0-blog-post-series]。

[LXD-introduction]: https://linuxcontainers.org/lxd/introduction/
[experience-lxd-in-ustack]: http://www.larrycaiyu.com/2015/07/14/experience-lxd-in-ustack.html
[try-lxd]: https://linuxcontainers.org/lxd/try-it/
[LXD-configuration]: https://github.com/lxc/lxd/blob/master/doc/configuration.md
[LXD-live-migration]: http://www.it-sobytie.ru/system/attachments/files/000/001/211/original/Tycho_Andersen_new.pdf?1479472850
[LXD-2-0-blog-post-series]: https://www.stgraber.org/2016/03/11/lxd-2-0-blog-post-series-012/
