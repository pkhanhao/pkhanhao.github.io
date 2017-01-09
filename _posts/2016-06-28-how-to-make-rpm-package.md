---
layout: post
title:  "如何制作rpm包"
date:   2016-06-28 17:33:00 +0800
categories: jekyll update
---

## 一、准备工作

### 1、安装工具包

    $ yum install fedora-packager rpmdevtools @Development Tools

### 2、建立非特权用户

    $ useradd -m -G mock makerpm
    $ id makerpm
    uid=1181(makerpm) gid=1181(makerpm) groups=1181(makerpm),135(mock)

### 3、建议保留文件时间戳

使用 wget 获取源代码，确保 ~/.wgetrc文 件包含此行 timestamping = on。

    $ echo "timestamping = on" >  ~/.wgetrc

使用 curl ，确保 ~/.curlrc 文件包含 -R 选项。

    $ echo "-R" >  ~/.curlrc

### 4、初始化打包目录

切换到 makerpm 账号，使用 rpmdev-setuptree 命令创建打包目录，程序将创建 ~/rpmbuild 以及一系列预设子目录。

    $ rpmdev-setuptree
    $ tree rpmbuild
    rpmbuild                   # 每个目录的用途后面会介绍到
    ├── BUILD
    ├── RPMS
    ├── SOURCES
    ├── SPECS
    └── SRPMS

rpmdev-setuptree 命令同时还会创建用于设置各种选项的 ~/.rpmmacros 文件。

    $ cat .rpmmacros
    %_topdir %(echo $HOME)/rpmbuild

    %_smp_mflags %( \
        [ -z "$RPM_BUILD_NCPUS" ] \\\
            && RPM_BUILD_NCPUS="`/usr/bin/nproc 2>/dev/null || \\\
                                 /usr/bin/getconf _NPROCESSORS_ONLN`"; \\\
        if [ "$RPM_BUILD_NCPUS" -gt 16 ]; then \\\
            echo "-j16"; \\\
        elif [ "$RPM_BUILD_NCPUS" -gt 3 ]; then \\\
            echo "-j$RPM_BUILD_NCPUS"; \\\
        else \\\
            echo "-j3"; \\\
        fi )

    %__arch_install_post \
        [ "%{buildarch}" = "noarch" ] || QA_CHECK_RPATHS=1 ; \
        case "${QA_CHECK_RPATHS:-}" in [1yY]*) /usr/lib/rpm/check-rpaths ;; esac \
        /usr/lib/rpm/check-buildroot

## 二、基础知识

构建一个标准的 rpm 包，需要创建 .spec 文件，它包含了软件打包的全部信息。了解 .spec 文件请参考[如何制作 RPM 包][How_to_create_an_RPM_package]中的“SPEC 文件剖析”部分。

对 .spec 文件执行 rpmbuild 打包命令（如 rpmbuild -ba NAME.spec），系统即按照特定的步骤生成最终 rpm 包。

    rpmbuild 参数说明：
       -bp   # 只执行 spec 的 %pre 阶段(解开源码包并打补丁，即只做准备)
       -bc   # 执行 spec 的 %pre 和 %build 阶段(准备并编译)
       -bi   # 执行 spec 中 %pre，%build 与 %install (准备，编译并安装)
       -bl   # 检查 spec 中 %file 阶段(查看文件是否齐全)
       -ba   # 建立源码与二进制包
       -bb   # 只建立二进制包
       -bs   # 只建立源码包

rpmbuild 命令读取 .spec 文件执行如下的步骤和阶段进行 rpm 包构建，表中以 % 开头的语句为预定义宏。

![build-phase](http://oiz97v6wa.bkt.clouddn.com/makerpm/build-phase.png)

上表中宏代码对应的目录关系如下：

![directory-role](http://oiz97v6wa.bkt.clouddn.com/makerpm/directory-role.png)

## 三、打包示例

1、这里简单示例构建 hello 的 rpm 包，细节可参考[如何创建一个 GNU Hello RPM 包][How_to_create_a_GNU_Hello_RPM_package]。

    $ cd ~/rpmbuild/SOURCES/
    $ wget http://ftp.gnu.org/gnu/hello/hello-2.10.tar.gz                # 获取源码包
    $ cd ~/rpmbuild/SPECS
    $ rpmdev-newspec hello
    $ vim hello.spec         # 编辑.spec文件，内容请参考“如何创建一个GNU Hello RPM包”
    $ rpmlint  hello.spec
    0 packages and 1 specfiles checked; 0 errors, 0 warnings.
    $ rpmbuild -bp  SPECS/hello.spec    # 依次执行，了解各阶段的具体动作，也可直接-ba
    $ rpmbuild -bc  SPECS/hello.spec
    $ rpmbuild -bi  SPECS/hello.spec
    $ rpmbuild -bl  SPECS/hello.spec
    $ rpmbuild -bs  SPECS/hello.spec
    $ rpmbuild -bb  SPECS/hello.spec
    $ rpmbuild -ba  SPECS/hello.spec
    $ tree . -L 3
    ├── RPMS
    │   └── x86_64
    │       ├── hello-2.10-1.el7.centos.x86_64.rpm
    │       └── hello-debuginfo-2.10-1.el7.centos.x86_64.rpm
    ├── SOURCES
    │   └── hello-2.10.tar.gz
    ├── SPECS
    │   └── hello.spec
    └── SRPMS
        └── hello-2.10-1.el7.centos.src.rpm
    $ rpmlint hello.spec ../RPMS/*/hello*.rpm ../SRPMS/hello*.rpm
    hello.x86_64: W: incoherent-version-in-changelog 2.10-1 ['2.10-1.el7.centos', '2.10-1.centos']
    hello.x86_64: W: file-not-utf8 /usr/share/doc/hello-2.10/THANKS
    3 packages and 1 specfiles checked; 0 errors, 2 warnings.

2、使用 [mock][Linux-mock] 构建不同于当前系统架构的 rpm 包。

mock 的配置文件在 /etc/mock/ 下，系统默认内置 fedora、epel 等系统的配置文件，通过查看这些 .cfg 文件，可以了解到 mock 的原理，即软件使用 yum 下载一个位于 /var/lib/mock/ 下的最小系统，并放入 chroot 环境中。

这里通过上一步构建的 hello-2.10-1.el7.centos.src.rpm 源码  rpm 包，重建 epel-6-x86_64 架构下的 rpm 包。

    $ /usr/bin/mock --verbose -r epel-6-x86_64 --rebuild ./SRPMS/hello-2.10-1.el7.centos.src.rpm
    $ ls /var/lib/mock/epel-6-x86_64/result/ -1
    build.log
    hello-2.10-1.el6.src.rpm
    hello-2.10-1.el6.x86_64.rpm
    hello-debuginfo-2.10-1.el6.x86_64.rpm
    root.log
    state.log

[How_to_create_an_RPM_package]: https://fedoraproject.org/wiki/How_to_create_an_RPM_package/zh-cn
[How_to_create_a_GNU_Hello_RPM_package]: https://fedoraproject.org/wiki/How_to_create_a_GNU_Hello_RPM_package/zh-cn
[Linux-mock]: https://www.zhukun.net/archives/6881
