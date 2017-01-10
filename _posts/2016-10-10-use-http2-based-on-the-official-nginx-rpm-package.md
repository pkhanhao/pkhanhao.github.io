---
layout: post
title:  "启用nginx HTTP/2"
date:   2016-10-10 18:30:00 +0800
categories: jekyll update
---

这里针对 nginx 1.10.1 静态编译[ openssl1.0.2j ][openssl]打第3方[ ChaCha20/Poly1305 path ][ChaCha20/Poly1305-path]，支持HTTP/2；并对 nginx 打[preread_buffer.patch][preread-buffer-patch]，解决[HTTP/2 POST Bug][http2-post-bug]。

### 一、获取 openssl 和 patch 文件

1、openssl1.0.2j + ChaCha20/Poly1305 path

    $ wget https://github.com/openssl/openssl/archive/OpenSSL_1_0_2j.tar.gz
    $ tar zxf OpenSSL_1_0_2j.tar.gz
    $ git clone  https://github.com/travislee8964/sslconfig.git
    $ cd openssl-OpenSSL_1_0_2j
    $ patch -p1 < ../sslconfig/patches/openssl__chacha20_poly1305_draft_and_rfc_ossl102i.patch
    $ mv openssl-OpenSSL_1_0_2j openssl-OpenSSL_1_0_2j_with_chacha20_poly1305
    $ tar czf openssl-OpenSSL_1_0_2j_with_chacha20_poly1305.tar.gz openssl-OpenSSL_1_0_2j_with_chacha20_poly1305

2、preread_buffer.patch。

    $ wget https://trac.nginx.org/nginx/raw-attachment/ticket/959/preread_buffer.patch

### 二、nginx rpm 扩展打包

下面会分2次打包：（1）用 openssl1.0.2j 编译的 nginx；（2）用 openssl1.0.2j 编译，并且运用 preread_buffer.patch 的 nginx 。

1、用 openssl1.0.2j 编译的 nginx 。

    $ ls -1  ../SOURCES/
    COPYRIGHT
    logrotate
    nginx-1.10.1.tar.gz
    nginx.conf
    nginx-debug.service
    nginx-debug.sysconf
    nginx.init.in
    nginx.service
    nginx.suse.logrotate
    nginx.sysconf
    nginx.upgrade.sh
    nginx.vh.default.conf
    njs-1c50334fbea6.tar.gz
    openssl-OpenSSL_1_0_2j_with_chacha20_poly1305.tar.gz
    preread_buffer.patch
    $ diff -u nginx.spec.origin nginx.spec
    diff --git a/nginx.spec b/nginx.spec
    index 8bec863..e0c2302 100644
    --- a/nginx.spec
    +++ b/nginx.spec
    @@ -61,7 +61,7 @@ BuildRequires: libGeoIP-devel
     # end of distribution specific definitions

     %define main_version                 1.10.1
    -%define main_release                 1%{?dist}.ngx
    +%define main_release                 1+pkhanhao2016101001%{?dist}.ngx
     %define module_xslt_version          %{main_version}
     %define module_xslt_release          1%{?dist}.ngx
     %define module_geoip_version         %{main_version}
    @@ -95,6 +95,7 @@ BuildRequires: libGeoIP-devel
             --user=%{nginx_user} \
             --group=%{nginx_group} \
    +        --with-openssl=openssl-OpenSSL_1_0_2j_with_chacha20_poly1305 \
             --with-http_ssl_module \
             --with-http_realip_module \
             --with-http_addition_module \
             --with-http_sub_module \
    @@ -143,6 +144,7 @@ Source10: nginx.suse.logrotate
     Source11: nginx-debug.service
     Source12: COPYRIGHT
     Source13: njs-%{module_njs_shaid}.tar.gz
    +Source14: openssl-OpenSSL_1_0_2j_with_chacha20_poly1305.tar.gz

     License: 2-clause BSD-like license

    @@ -211,6 +213,7 @@ Dynamic nJScript module for nginx.
     %prep
     %setup -q
     tar xvzf %SOURCE13
    +tar xvzf %SOURCE14
     cp %{SOURCE2} .
     sed -e 's|%%DEFAULTSTART%%|2 3 4 5|g' -e 's|%%DEFAULTSTOP%%|0 1 6|g' \
         -e 's|%%PROVIDES%%|nginx|g' < %{SOURCE2} > nginx.init
    @@ -560,6 +563,12 @@ if [ $1 -ge 1 ]; then
     fi
     %changelog
    +* Tue Oct 10 2016 pkhanhao <pkhanhao@163.com>
    +- 1.10.1+pkhanhao2016101001
    +- Use openssl-OpenSSL_1_0_2j_with_chacha20_poly1305
    +- openssl-OpenSSL_1_0_2j: https://github.com/openssl/openssl/archive/OpenSSL_1_0_2j.tar.gz
    +- chacha20_poly1305 patch: https://github.com/travislee8964/sslconfig.git
    +
     * Tue May 31 2016 Konstantin Pavlov <thresh@nginx.com>
     - 1.10.1
    $ rpmbuild -bi nginx.spec
    $ rpmbuild -bl nginx.spec
    $ rpmbuild -ba nginx.spec

2、用 openssl1.0.2j 编译，并且运用 preread_buffer.patch 的 nginx 。

    $ diff -u nginx.spec.origin nginx.spec
    diff --git a/nginx.spec b/nginx.spec
    index e0c2302..c5fa1ae 100644
    --- a/nginx.spec
    +++ b/nginx.spec
    @@ -61,7 +61,7 @@ BuildRequires: libGeoIP-devel
     # end of distribution specific definitions

     %define main_version                 1.10.1
    -%define main_release                 1+pkhanhao2016101001%{?dist}.ngx
    +%define main_release                 1+pkhanhao2016101002%{?dist}.ngx
     %define module_xslt_version          %{main_version}
     %define module_xslt_release          1%{?dist}.ngx
     %define module_geoip_version         %{main_version}
    @@ -146,6 +146,8 @@ Source12: COPYRIGHT
     Source13: njs-%{module_njs_shaid}.tar.gz
     Source14: openssl-OpenSSL_1_0_2j_with_chacha20_poly1305.tar.gz

    +Patch0: preread_buffer.patch
    +
     License: 2-clause BSD-like license

     BuildRoot: %{_tmppath}/%{name}-%{main_version}-%{main_release}-root
    @@ -219,6 +221,7 @@ sed -e 's|%%DEFAULTSTART%%|2 3 4 5|g' -e 's|%%DEFAULTSTOP%%|0 1 6|g' \
         -e 's|%%PROVIDES%%|nginx|g' < %{SOURCE2} > nginx.init
     sed -e 's|%%DEFAULTSTART%%||g' -e 's|%%DEFAULTSTOP%%|0 1 2 3 4 5 6|g' \
         -e 's|%%PROVIDES%%|nginx-debug|g' < %{SOURCE2} > nginx-debug.init
    +%patch -P 0 -p1                      # 使用第0号Patch文件，参数为-p1

     %build
     ./configure %{COMMON_CONFIGURE_ARGS} \
    @@ -564,6 +567,11 @@ fi

     %changelog
    +* Tue Oct 10 2016 pkhanhao <pkhanhao@163.com>
    +- 1.10.1+pkhanhao2016101002
    +- Use https://trac.nginx.org/nginx/ticket/959 patch
    +- preread_buffer.patch: https://trac.nginx.org/nginx/raw-attachment/ticket/959/preread_buffer.patch
    +
     * Tue Oct 10 2016 pkhanhao <pkhanhao@163.com>
     - 1.10.1+pkhanhao2016101001
     - Use openssl-OpenSSL_1_0_2j_with_chacha20_poly1305
     - openssl-OpenSSL_1_0_2j: https://github.com/openssl/openssl/archive/OpenSSL_1_0_2j.tar.gz

    $ rpmbuild -bi nginx.spec
    $ rpmbuild -bl nginx.spec
    $ rpmbuild -ba nginx.spec
    $ ls -1 ../SRPMS/
    nginx-1.10.1-1+pkhanhao2016101001.el7.centos.ngx.src.rpm
    nginx-1.10.1-1+pkhanhao2016101002.el7.centos.ngx.src.rpm

### 三、CentOS 6.x 下打包遇到的问题

CentOS 6.x 使用 rpmbuild 或 mock 构建 epel-6-x86_64 架构下的 rpm 包时，报错：

    poly1305_avx2.s: Assembler messages:
    poly1305_avx2.s:335: Error: suffix or operands invalid for `vpunpcklqdq'
    poly1305_avx2.s:336: Error: suffix or operands invalid for `vpunpckhqdq'
    poly1305_avx2.s:338: Error: no such instruction: `vpermq $0xD8,%ymm7,%ymm7'
    poly1305_avx2.s:339: Error: no such instruction: `vpermq $0xD8,%ymm8,%ymm8'
    poly1305_avx2.s:341: Error: suffix or operands invalid for `vpsrlq'
    poly1305_avx2.s:342: Error: suffix or operands invalid for `vpand'
    poly1305_avx2.s:343: Error: suffix or operands invalid for `vpaddq'
    poly1305_avx2.s:345: Error: suffix or operands invalid for `vpsrlq'
    poly1305_avx2.s:346: Error: suffix or operands invalid for `vpand'
    poly1305_avx2.s:347: Error: suffix or operands invalid for `vpaddq'
    poly1305_avx2.s:349: Error: suffix or operands invalid for `vpsllq'
    poly1305_avx2.s:350: Error: suffix or operands invalid for `vpxor'
    poly1305_avx2.s:351: Error: suffix or operands invalid for `vpand'
    poly1305_avx2.s:352: Error: suffix or operands invalid for `vpaddq'
    poly1305_avx2.s:354: Error: suffix or operands invalid for `vpsrlq'
    poly1305_avx2.s:355: Error: suffix or operands invalid for `vpsrlq'
    poly1305_avx2.s:356: Error: suffix or operands invalid for `vpand'
    poly1305_avx2.s:357: Error: suffix or operands invalid for `vpxor'
    poly1305_avx2.s:358: Error: suffix or operands invalid for `vpaddq'
    poly1305_avx2.s:359: Error: suffix or operands invalid for `vpaddq'
    poly1305_avx2.s:362: Error: no such instruction: `vpbroadcastq 64(%rdi),%ymm5'
    ......
    poly1305_avx2.s:784: Error: no such instruction: `vpermq $0xAA,%ymm0,%ymm7'
    poly1305_avx2.s:785: Error: no such instruction: `vpermq $0xAA,%ymm1,%ymm8'
    poly1305_avx2.s:786: Error: no such instruction: `vpermq $0xAA,%ymm2,%ymm9'
    poly1305_avx2.s:787: Error: no such instruction: `vpermq $0xAA,%ymm3,%ymm10'
    poly1305_avx2.s:788: Error: no such instruction: `vpermq $0xAA,%ymm4,%ymm11'
    poly1305_avx2.s:790: Error: suffix or operands invalid for `vpaddq'
    poly1305_avx2.s:791: Error: suffix or operands invalid for `vpaddq'
    poly1305_avx2.s:792: Error: suffix or operands invalid for `vpaddq'
    poly1305_avx2.s:793: Error: suffix or operands invalid for `vpaddq'
    poly1305_avx2.s:794: Error: suffix or operands invalid for `vpaddq'
    make[4]: Leaving directory `/builddir/build/BUILD/nginx-1.10.1/openssl-OpenSSL_1_0_2j_with_chacha20_poly1305/crypto/chacha20poly1305'
    make[3]: Leaving directory `/builddir/build/BUILD/nginx-1.10.1/openssl-OpenSSL_1_0_2j_with_chacha20_poly1305/crypto'
    make[2]: Leaving directory `/builddir/build/BUILD/nginx-1.10.1/openssl-OpenSSL_1_0_2j_with_chacha20_poly1305'
    make[1]: Leaving directory `/builddir/build/BUILD/nginx-1.10.1'

    RPM build errors:
    make[4]: *** [poly1305_avx2.o] Error 1
    make[3]: *** [subdirs] Error 1
    make[2]: *** [build_crypto] Error 1
    make[1]: *** [openssl-OpenSSL_1_0_2j_with_chacha20_poly1305/.openssl/include/openssl/ssl.h] Error 2
    make: *** [build] Error 2

根据[这里][sslconfig-issues]的信息提示需要使用高版本 gcc。使用 mock 查看 epel-6-x86_64 架构，gcc 也是低版本。

    $ /usr/bin/mock -r epel-6-x86_64 --chroot "gcc --version"
    gcc version 4.4.7 20120313 (Red Hat 4.4.7-17)
    $ /usr/bin/mock -r epel-7-x86_64 --chroot "gcc --version"
    gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-4)

这里选择使用[ devtoolset-3 ][devtoolset-3]，不对系统本身的 gcc 做操作影响。

    $ yum install centos-release-scl
    $ yum install devtoolset-3-gcc              # 获取devtoolset-3的gcc
    $ ls -1 /opt/rh/devtoolset-3/
    enable
    root
    $ scl enable devtoolset-3 bash              # 临时获取和激活devtoolset-3 bash环境
    $ echo "source /opt/rh/devtoolset-3/enable" >> ~/.bashrc   # 如需要，可设环境变量，永久激活
    $ echo $PATH
    /opt/rh/devtoolset-3/root/usr/bin:/usr/lib64/qt-3.3/bin:/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/sbin:/home/makerpm/bin
    $ which  gcc
    /opt/rh/devtoolset-3/root/usr/bin/gcc
    $ gcc -v
    gcc version 4.9.2 20150212 (Red Hat 4.9.2-6) (GCC)
    $ rpmbuild -bi nginx.spec
    $ rpmbuild -bl nginx.spec
    $ rpmbuild -ba nginx.spec

### 四、nginx 配置 HTTP/2

nginx 虚拟主机开启 ssl ，并配置使用[ HTTP/2 ][http2-resource]。

    listen 443 ssl http2;
    ......
    ssl                  on;
    ssl_certificate      /srv/cert/server.crt;
    ssl_certificate_key  /srv/cert/server.key;
    ssl_session_timeout  5m;
    ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers  EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
    ssl_prefer_server_ciphers   on;

使用的 ssl 协议版本和对应算法。

    $ openssl ciphers -V 'EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256ECDH+3DES:RSA+3DES:!MD5' | column -t | grep TLSv1.2
    0xC0,0x2F  -  ECDHE-RSA-AES128-GCM-SHA256    TLSv1.2  Kx=ECDH  Au=RSA    Enc=AESGCM(128)  Mac=AEAD
    0xC0,0x2B  -  ECDHE-ECDSA-AES128-GCM-SHA256  TLSv1.2  Kx=ECDH  Au=ECDSA  Enc=AESGCM(128)  Mac=AEAD
    0xC0,0x27  -  ECDHE-RSA-AES128-SHA256        TLSv1.2  Kx=ECDH  Au=RSA    Enc=AES(128)     Mac=SHA256
    0xC0,0x23  -  ECDHE-ECDSA-AES128-SHA256      TLSv1.2  Kx=ECDH  Au=ECDSA  Enc=AES(128)     Mac=SHA256
    0x00,0x9C  -  AES128-GCM-SHA256              TLSv1.2  Kx=RSA   Au=RSA    Enc=AESGCM(128)  Mac=AEAD
    0x00,0x3C  -  AES128-SHA256                  TLSv1.2  Kx=RSA   Au=RSA    Enc=AES(128)     Mac=SHA256
    0xC0,0x30  -  ECDHE-RSA-AES256-GCM-SHA384    TLSv1.2  Kx=ECDH  Au=RSA    Enc=AESGCM(256)  Mac=AEAD
    0xC0,0x2C  -  ECDHE-ECDSA-AES256-GCM-SHA384  TLSv1.2  Kx=ECDH  Au=ECDSA  Enc=AESGCM(256)  Mac=AEAD
    0xC0,0x28  -  ECDHE-RSA-AES256-SHA384        TLSv1.2  Kx=ECDH  Au=RSA    Enc=AES(256)     Mac=SHA384
    0xC0,0x24  -  ECDHE-ECDSA-AES256-SHA384      TLSv1.2  Kx=ECDH  Au=ECDSA  Enc=AES(256)     Mac=SHA384
    0x00,0x9D  -  AES256-GCM-SHA384              TLSv1.2  Kx=RSA   Au=RSA    Enc=AESGCM(256)  Mac=AEAD
    0x00,0x3D  -  AES256-SHA256                  TLSv1.2  Kx=RSA   Au=RSA    Enc=AES(256)     Mac=SHA256

### 五、nginx 配置 HTTP/2 遇到的问题

通过谷歌、火狐浏览器访问配置了 http2 的特定域名，均报错“ Server negotiated HTTP/2 with protocol version lower than 1.2 ”。使用[ SSL LABS ][SSL-LABS]或[ testssl 脚本][testssl]（如 testssl.sh   dev.testssl.sh）验证显示：不提供/支持 TLS1.1 和 1.2（但提供/支持 ALPN 协商协议）；通过 Wireshark 抓包也发现 hello 包最终协商使用的 TLS1.0。

排查思路：此 nginx 实例存在多个域名，同时对外提供 http/https 服务，怀疑是多个虚拟主机配置存在先后顺序的关系，即被主（默认）配置文件中的定义影响。于是尝试删除其他虚拟主机配置文件，重启服务，特定域名 http2 访问正常。

解决方法：443端口同时监听多个 https 虚拟主机服务，顺序位靠前的虚拟主机 ssl 版本最高支持到 TLS1.0 ，从而影响了 http2 协议导致报错。更新所有 https 服务 ssl 版本和加密套件算法，问题解决。

[openssl]: https://github.com/openssl/openssl
[ChaCha20/Poly1305-path]: https://github.com/travislee8964/sslconfig
[http2-post-bug]: https://imququ.com/post/nginx-http2-post-bug.html
[preread-buffer-patch]: https://trac.nginx.org/nginx/ticket/959
[sslconfig-issues]: https://github.com/cloudflare/sslconfig/issues/32
[devtoolset-3]: https://gist.github.com/stephenturner/e3bc5cfacc2dc67eca8b
[http2-resource]: https://imququ.com/post/http2-resource.html
[SSL-LABS]: https://www.ssllabs.com/ssltest/index.html
[testssl]: https://testssl.sh/
