---
layout: post
title:  "nginx rpm包增加后端健康检测模块"
date:   2016-06-29 18:20:00 +0800
categories: jekyll update
---

## 一、获取源码 rpm 包

1、rpm 包有2种：二进制 rpm 包 和 源码 rpm 包。

二进制 rpm 包：形如 xxx.rpm ，用于系统软件安装，包含已完成编译的可执行文件和相关配置文件。

    $ rpm -qlp nginx-1.10.1-1.el7.ngx.x86_64.rpm
    /etc/logrotate.d/nginx
    /etc/nginx
    /etc/nginx/conf.d
    /etc/nginx/conf.d/default.conf
    /etc/nginx/fastcgi_params
    /etc/nginx/koi-utf
    /etc/nginx/koi-win
    /etc/nginx/mime.types
    /etc/nginx/modules
    /etc/nginx/nginx.conf
    /etc/nginx/scgi_params
    /etc/nginx/uwsgi_params
    /etc/nginx/win-utf
    /etc/sysconfig/nginx
    /etc/sysconfig/nginx-debug
    /usr/lib/systemd/system/nginx-debug.service
    /usr/lib/systemd/system/nginx.service
    /usr/lib64/nginx
    /usr/lib64/nginx/modules
    /usr/libexec/initscripts/legacy-actions/nginx
    /usr/libexec/initscripts/legacy-actions/nginx/upgrade
    /usr/sbin/nginx
    /usr/sbin/nginx-debug
    /usr/share/doc/nginx-1.10.1
    /usr/share/doc/nginx-1.10.1/COPYRIGHT
    /usr/share/nginx
    /usr/share/nginx/html
    /usr/share/nginx/html/50x.html
    /usr/share/nginx/html/index.html
    /var/cache/nginx
    /var/log/nginx

源码 rpm 包：形如 xxx.src.rpm ，用于开源软件发布源码，一般包含 xxx.spec 文件和 xxx.tar.gz 文件等。

    $ rpm -qlp nginx-1.10.1-1.el7.ngx.src.rpm
    COPYRIGHT
    logrotate
    nginx-1.10.1.tar.gz
    nginx-debug.service
    nginx-debug.sysconf
    nginx.conf
    nginx.init.in
    nginx.service
    nginx.spec
    nginx.suse.logrotate
    nginx.sysconf
    nginx.upgrade.sh
    nginx.vh.default.conf
    njs-1c50334fbea6.tar.gz

2、获取 nginx rpm 包。

设置 nginx 官方 yum 源，用于获取相应的 rpm 包。

    $ cat /etc/yum.repos.d/nginx.repo
    [nginx-Stable]
    name=nginx Stable
    baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
    gpgcheck=0
    enabled=1
    [nginx-SRPMS]
    name=nginx SRPMS
    baseurl=http://nginx.org/packages/centos/$releasever/SRPMS/
    gpgcheck=0
    enabled=1

方法一：

以非安装方式获取二进制 rpm 包，如出现“ no such option: --downloadonly ”需先升级 yum 包本身：yum install yum。

    $ yum install --downloadonly --downloaddir=./ nginx

不指定 --downloaddir 参数，默认获取到 /var/cache/yum/x86_64/[OsVersion]/[Repository]/packages 目录下。

    $ ls /var/cache/yum/x86_64/7/nginx/packages/

获取源码 rpm 包：

    $ wget http://nginx.org/packages/centos/7/SRPMS/nginx-1.10.1-1.el7.ngx.src.rpm

方法二[推荐]：

以非root权限，使用 yumdownloader 命令直接获取2种rpm包。

    $ yumdownloader nginx
    $ yumdownloader --source nginx

3、反解 rpm 包

这里主要关注和打包有关的源码 rpm 包。

    $ rpm2cpio nginx-1.10.1-1.el7.ngx.src.rpm > nginx.src.cpio
    $ cpio -idmv < nginx.src.cpio

或：

    $ rpm2cpio nginx-1.10.1-1.el7.ngx.src.rpm | cpio -idmv

查看解包后的资源，其中最重要的是 nginx.spec 文件。

    $ ls -1
    COPYRIGHT
    logrotate
    nginx-1.10.1.tar.gz
    nginx.conf
    nginx-debug.service
    nginx-debug.sysconf
    nginx.init.in
    nginx.service
    nginx.spec
    nginx.suse.logrotate
    nginx.sysconf
    nginx.upgrade.sh
    nginx.vh.default.conf
    njs-1c50334fbea6.tar.gz

## 二、rpm 包添加后端健康检测模块

这里我们将针对 nginx 加入[后端健康检测][yaoweibin_nginx_upstream_check_module]模块。

1、清空 SOURCES 下的 tar 包和 patch 文件等。

    $ rm -fr SOURCES/*

2、将源码 rpm 包反解得来的源文件等放入 SOURCES 下，并把 nginx.spec 文件放入 SPECS 下。

    $ rpm -ivh nginx-1.10.1-1.el7.ngx.src.rpm    # 不解包可直接使用rpm命令达到目的

3、获取检测模块。注意，这里的 f3bdb7b 是指当前 master 分支上最后一个 commit id 。

    $ wget https://codeload.github.com/yaoweibin/nginx_upstream_check_module/zip/master \
    -O nginx_upstream_check_module-f3bdb7b.zip
    $ unzip nginx_upstream_check_module-f3bdb7b.zip
    $ ls -1
    CHANGES
    check_1.11.1+.patch
    check_1.2.1.patch
    check_1.2.2+.patch
    check_1.2.6+.patch
    check_1.5.12+.patch
    check_1.7.2+.patch
    check_1.7.5+.patch
    check_1.9.2+.patch
    check.patch
    config
    doc
    nginx-sticky-module.patch
    nginx-tests
    ngx_http_upstream_check_module.c
    ngx_http_upstream_check_module.h
    ngx_http_upstream_jvm_route_module.patch
    README
    test
    upstream_fair.patch
    util

README 文件中有详细的安装、使用方法。

    $ cat README
    Installation
        Download the latest version of the release tarball of this module from
        github (<http://github.com/yaoweibin/nginx_upstream_check_module>)

        Grab the nginx source code from nginx.org (<http://nginx.org/>), for
        example, the version 1.0.14 (see nginx compatibility), and then build
        the source with this module:

    $ wget 'http://nginx.org/download/nginx-1.0.14.tar.gz'
    $ tar -xzvf nginx-1.0.14.tar.gz
    $ cd nginx-1.0.14/
    $ patch -p1 < /path/to/nginx_http_upstream_check_module/check.patch
    $ ./configure --add-module=/path/to/nginx_http_upstream_check_module
    $ make
    $ make install

**思路梳理**：如通常的 tar 包源码编译方法，（1）对 nginx 实施特定 patch ；（2）增加模块编译参数，指定模块包所在目录。

4、加料 SOURCES。这里选取check_1.9.2+.patch，因为我们使用的nginx版本为1.10.x，即1.9.x的release版本。

    $ cp nginx_upstream_check_module-f3bdb7b.zip rpmbuild/SOURCES/
    $ cp nginx_upstream_check_module-master/check_1.9.2+.patch rpmbuild/SOURCES/

5、修改nginx.spec。

    $ cp nginx.spec nginx.spec.origin               # 备份先
    $ vim nginx.spec
    $ diff -u nginx.spec.origin nginx.spec          # diff 对比
    --- nginx.spec.origin
    +++ nginx.spec
    @@ -61,7 +61,7 @@
     # end of distribution specific definitions

     %define main_version                 1.10.1

修改软件[版本号][RPM-version]，这里在上游版本基础上增加了小版本。

    -%define main_release                 1%{?dist}.ngx
    +%define main_release                 1+pkhanhao20160629%{?dist}.ngx    # 增加小版本号
     %define module_xslt_version          %{main_version}
     %define module_xslt_release          1%{?dist}.ngx
     %define module_geoip_version         %{main_version}
    @@ -73,6 +73,7 @@
     %define module_njs_shaid             1c50334fbea6
     %define module_njs_version           %{main_version}.0.0.20160414.%{module_njs_shaid}
     %define module_njs_release           1%{?dist}.ngx
    +%define module_upstream_check_commitid             f3bdb7b             # 定义模块commit id变量

     %define bdir %{_builddir}/%{name}-%{main_version}

    @@ -112,6 +113,7 @@
             --with-http_geoip_module=dynamic \
             --with-http_perl_module=dynamic \
             --add-dynamic-module=njs-%{module_njs_shaid}/nginx \
    +        --add-module=nginx_upstream_check_module-master \              # 增加模块编译参数
             --with-threads \
             --with-stream \
             --with-stream_ssl_module \
    @@ -143,6 +145,9 @@
     Source11: nginx-debug.service
     Source12: COPYRIGHT
     Source13: njs-%{module_njs_shaid}.tar.gz
    +Source14: nginx_upstream_check_module-%{module_upstream_check_commitid}.zip   # 定义模块名
    +
    +Patch0: check_1.9.2+.patch              # 定义补丁文件名

     License: 2-clause BSD-like license

    @@ -211,11 +216,13 @@
     %prep                                   # prep 阶段 #
     %setup -q
     tar xvzf %SOURCE13
    +unzip %SOURCE14                         # 解包
     cp %{SOURCE2} .
     sed -e 's|%%DEFAULTSTART%%|2 3 4 5|g' -e 's|%%DEFAULTSTOP%%|0 1 6|g' \
         -e 's|%%PROVIDES%%|nginx|g' < %{SOURCE2} > nginx.init
     sed -e 's|%%DEFAULTSTART%%||g' -e 's|%%DEFAULTSTOP%%|0 1 2 3 4 5 6|g' \
         -e 's|%%PROVIDES%%|nginx-debug|g' < %{SOURCE2} > nginx-debug.init

[打patch][RPM-macros-patch]，即 patch -p0  < /path/SOURCES/check_1.9.2+.patch 。

    +%patch0                                 # 打补丁

     %build
     ./configure %{COMMON_CONFIGURE_ARGS} \
    @@ -560,6 +567,10 @@
     fi

     %changelog
    +* Tue Jun 29 2016 pkhanhao <pkhanhao@163.com>                   # 更新日志记录
    +- 1.10.1+pkhanhao20160629
    +- Add upstream_check_module.
    +
     * Tue May 31 2016 Konstantin Pavlov <thresh@nginx.com>
     - 1.10.1

6、打包和查验打包过程。

    $ rpmbuild -bp nginx.spec
    $ ls -1 ../BUILD/nginx-1.10.1/
    auto
    CHANGES
    CHANGES.ru
    conf
    configure
    contrib
    html
    LICENSE
    man
    nginx-debug.init
    nginx.init
    nginx.init.in
    nginx_upstream_check_module-master
    njs-1c50334fbea6
    README
    src
    $ rpmbuild -bc nginx.spec
    $ ls -1 ../BUILD/nginx-1.10.1/
    auto
    CHANGES
    CHANGES.ru
    conf
    configure
    contrib
    html
    LICENSE
    Makefile
    man
    nginx-debug.init
    nginx.init
    nginx.init.in
    nginx_upstream_check_module-master
    njs-1c50334fbea6
    objs
    README
    src
    $ ../BUILD/nginx-1.10.1/objs/nginx -V
    nginx version: nginx/1.10.1
    built by gcc 4.8.5 20150623 (Red Hat 4.8.5-4) (GCC)
    built with OpenSSL 1.0.1e-fips 11 Feb 2013
    TLS SNI support enabled
    configure arguments: ... --add-module=nginx_upstream_check_module-master ...
    $ rpmbuild -bi nginx.spec
    $ ls -1 ../BUILDROOT/nginx-1.10.1-1+pkhanhao20160629.el7.centos.ngx.x86_64/
    etc
    usr
    var
    $ rpmbuild -bs nginx.spec
    $ ls ../SRPMS/
    nginx-1.10.1-1+pkhanhao20160629.el7.centos.ngx.src.rpm
    $ rpmbuild -bb nginx.spec
    $ ls ../RPMS/x86_64/nginx*
    ../RPMS/x86_64/nginx-1.10.1-1+pkhanhao20160629.el7.centos.ngx.x86_64.rpm
    ../RPMS/x86_64/nginx-debuginfo-1.10.1-1+pkhanhao20160629.el7.centos.ngx.x86_64.rpm
    ../RPMS/x86_64/nginx-module-geoip-1.10.1-1.el7.centos.ngx.x86_64.rpm
    ../RPMS/x86_64/nginx-module-image-filter-1.10.1-1.el7.centos.ngx.x86_64.rpm
    ../RPMS/x86_64/nginx-module-njs-1.10.1.0.0.20160414.1c50334fbea6-1.el7.centos.ngx.x86_64.rpm
    ../RPMS/x86_64/nginx-module-perl-1.10.1-1.el7.centos.ngx.x86_64.rpm
    ../RPMS/x86_64/nginx-module-xslt-1.10.1-1.el7.centos.ngx.x86_64.rpm
    $ rpmlint nginx.spec ../SRPMS/nginx* ../RPMS/*/nginx*

7、使用[mock][Linux-mock]构建其他架构的rpm包。

    $ /usr/bin/mock  --verbose -r epel-6-x86_64 --rebuild \
    ../SRPMS/nginx-1.10.1-1+pkhanhao20160629.el7.centos.ngx.src.rpm
    $ ls -1 /var/lib/mock/epel-6-x86_64/result/
    build.log
    nginx-1.10.1-1+pkhanhao20160629.el6.ngx.src.rpm
    nginx-1.10.1-1+pkhanhao20160629.el6.ngx.x86_64.rpm
    nginx-debuginfo-1.10.1-1+pkhanhao20160629.el6.ngx.x86_64.rpm
    nginx-module-geoip-1.10.1-1.el6.ngx.x86_64.rpm
    nginx-module-image-filter-1.10.1-1.el6.ngx.x86_64.rpm
    nginx-module-njs-1.10.1.0.0.20160414.1c50334fbea6-1.el6.ngx.x86_64.rpm
    nginx-module-perl-1.10.1-1.el6.ngx.x86_64.rpm
    nginx-module-xslt-1.10.1-1.el6.ngx.x86_64.rpm
    root.log
    state.log

[yaoweibin_nginx_upstream_check_module]: https://github.com/yaoweibin/nginx_upstream_check_module
[RPM-version]: http://blog.csdn.net/rhel_admin/article/details/37592971
[RPM-macros-patch]: http://rpm.org/max-rpm-snapshot/s1-rpm-inside-macros.html
[Linux-mock]: https://www.zhukun.net/archives/6881
