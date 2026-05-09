# Python2.7环境"防腐"(Preservation)
```shell
# 在centos7上
[root@localhost ~]# python2 --version
Python 2.7.5

适用场景：Rocky9已经彻底删除了Python 2，但客户有个科研脚本或老旧工具必须依赖Python 2.7及其特定版本的C扩展库
1. 策略：构建“离散式”运行环境
不要试图在 Rocky 9 系统层安装Python 2，要采用完全隔离的目录安装

# 提取基础环境
最好的办法是从一台CentOS7上打包现成的环境：
# 在CentOS 7上执行
[root@localhost ~]# tar -czvf python27_base.tar.gz  /usr/bin/python2.7 /usr/lib64/python2.7  /usr/include/python2.7
[root@localhost ~]# ls -alh python27_base.tar.gz 
-rw-r--r-- 1 root root 7.7M 5月   8 19:57 python27_base.tar.gz


# 发送到rocky9上
[root@localhost ~]# scp python27_base.tar.gz root@172.16.186.140:~



# 在rocky9上
[root@localhost ~]# mkdir -p /opt/py2_legacy

在Rocky9上创建一个专门放 Pytho 2依赖库的文件夹，避免与系统 /lib64 混淆：
[root@localhost ~]# mkdir -p /opt/py2_legacy/libs
解压包，注意这里不是解压到根
[root@localhost ~]# tar -xzvf python27_base.tar.gz -C /opt/py2_legacy/

# 完整地找到缺失的库
[root@rocky9 ~]# ldd /opt/py2_legacy/usr/bin/python2.7
	linux-vdso.so.1 (0x00007ffc1d4ce000)
	libpython2.7.so.1.0 => not found
	libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f2509c16000)
	libdl.so.2 => /lib64/libdl.so.2 (0x00007f2509c11000)
	libutil.so.1 => /lib64/libutil.so.1 (0x00007f2509c0c000)
	libm.so.6 => /lib64/libm.so.6 (0x00007f2509b31000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f2509800000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f2509c21000)
注意：
你会看到类似 libpython2.7.so.1.0 => not found 的输出

# 在centos7上找到
[root@localhost ~]# find /usr/lib64 -name "libpython2.7.so.1.0"
/usr/lib64/libpython2.7.so.1.0
[root@localhost ~]# scp /usr/lib64/libpython2.7.so.1.0 root@172.16.186.140:/opt/py2_legacy/libs         # 140是rocky9的IP


Python很多功能(如SSL,MySQL,XML)是靠.so格式的扩展模块实现的。也需要检查它们：
# 扫描所有 Python 扩展模块是否缺库
[root@rocky9 ~]# find /opt/py2_legacy/usr/lib64/python2.7/lib-dynload/ -name "*.so" | xargs ldd | grep "not found"
	libnsl.so.1 => not found
	libpython2.7.so.1.0 => not found
	libpanelw.so.5 => not found
	libncursesw.so.5 => not found
	libtinfo.so.5 => not found
	libpython2.7.so.1.0 => not found
	libssl.so.10 => not found
	libcrypto.so.10 => not found
	libpython2.7.so.1.0 => not found
	libffi.so.6 => not found
	libpython2.7.so.1.0 => not found
	libpython2.7.so.1.0 => not found
	libpython2.7.so.1.0 => not found
	libgdbm.so.4 => not found
	libreadline.so.6 => not found
	libtinfo.so.5 => not found
	libpython2.7.so.1.0 => not found
	libssl.so.10 => not found
	libcrypto.so.10 => not found
	libpython2.7.so.1.0 => not found
	libpython2.7.so.1.0 => not found
	libgdbm.so.4 => not found
	libpython2.7.so.1.0 => not found
	libncursesw.so.5 => not found
	libtinfo.so.5 => not found
	libpython2.7.so.1.0 => not found
	libpython2.7.so.1.0 => not found
	libpython2.7.so.1.0 => not found
	libpython2.7.so.1.0 => not found
	libpython2.7.so.1.0 => not found
	libpython2.7.so.1.0 => not found
	libpython2.7.so.1.0 => not found
	libpython2.7.so.1.0 => not found
	libpython2.7.so.1.0 => not found

如果有输出，继续重复“从旧系统找库 -> 搬运到 /opt/py2_legacy/libs”的过程


# 过滤掉 not found 字样，只留下纯粹的文件名，并排掉重复项
[root@rocky9 ~]# find /opt/py2_legacy/usr/lib64/python2.7/lib-dynload/ -name "*.so" | xargs ldd 2>/dev/null | grep "not found" | awk '{print $1}' | sort | uniq
libcrypto.so.10
libffi.so.6
libgdbm.so.4
libncursesw.so.5
libnsl.so.1
libpanelw.so.5
libpython2.7.so.1.0
libreadline.so.6
libssl.so.10
libtinfo.so.5


# 回到centos7上找这些库
[root@localhost ~]# vim need_libs.txt
libcrypto.so.10
libffi.so.6
libgdbm.so.4
libncursesw.so.5
libnsl.so.1
libpanelw.so.5
libpython2.7.so.1.0
libreadline.so.6
libssl.so.10
libtinfo.so.5
libdbus-glib-1.so.2
liblua-5.1.so
libnss3.so
librpmbuild.so.3
librpmio.so.3
librpmsign.so.1
librpm.so.3
libnssutil3.so
libplc4.so
libplds4.so
libnspr4.so
liblzma.so.5
libselinux.so.1
libsepol.so.1
libnssutil3.so
libsmime3.so
libssl3.so
libfreebl3.so
libnspr4.so
libplc4.so
libplds4.so
libpopt.so.0
libdb-5.3.so
libnss3.so
libnssutil3.so
libsmime3.so
libssl3.so
libsoftokn3.so
libsoftokn3.chk
libfreebl3.so
libfreebl3.chk
libfreeblpriv3.so
libfreeblpriv3.chk
libnspr4.so
libplc4.so
libplds4.so



[root@localhost ~]# yum install mlocate -y
[root@localhost ~]# updatedb


# 定义你刚才在Rocky9看到的缺失库列表
MISSING_LIBS="libnsl.so.1 libpython2.7.so.1.0 libpanelw.so.5 libncursesw.so.5 libtinfo.so.5
libssl.so.10 libcrypto.so.10 libffi.so.6 libgdbm.so.4 libreadline.so.6
libssl.so.10
libtinfo.so.5
libdbus-glib-1.so.2
liblua-5.1.so
libnss3.so
librpmbuild.so.3
librpmio.so.3
librpmsign.so.1
librpm.so.3
libnssutil3.so
libplc4.so
libplds4.so
libnspr4.so
liblzma.so.5
libselinux.so.1
libsepol.so.1
libnssutil3.so
libsmime3.so
libssl3.so
libfreebl3.so
libnspr4.so
libplc4.so
libplds4.so
libpopt.so.0
libdb-5.3.so
libnss3.so
libnssutil3.so
libsmime3.so
libssl3.so
libsoftokn3.so
libsoftokn3.chk
libfreebl3.so
libfreebl3.chk
libfreeblpriv3.so
libfreeblpriv3.chk
libnspr4.so
libplc4.so
libplds4.so
"

[root@localhost ~]# mkdir -p /tmp/py27_libs_collect
[root@localhost ~]# for lib in $(cat need_libs.txt); do
    path=$(find /usr/lib64 /lib64 /usr/lib /lib -name "$lib" 2>/dev/null | head -n 1)
    if [ -n "$path" ]; then
        cp -L "$path" /tmp/py27_libs_collect/
    fi
done

[root@localhost ~]# ls /tmp/py27_libs_collect
libcrypto.so.10  libgdbm.so.4      libnsl.so.1     libpython2.7.so.1.0  libssl.so.10
libffi.so.6      libncursesw.so.5  libpanelw.so.5  libreadline.so.6     libtinfo.so.5


# 打包
[root@localhost ~]# tar -czvf py27_missing_libs.tar.gz -C /tmp/py27_libs_collect .

# 发送到rocky9上
[root@localhost ~]# scp py27_missing_libs.tar.gz root@172.16.186.140:~


# 在rocky9上解压到/opt/py2_legacy/libs目录下
[root@rocky9 ~]# tar -zxvf py27_missing_libs.tar.gz -C /opt/py2_legacy/libs

给所有搬过来的库和扩展赋予执行权限
[root@rocky9 ~]# chmod +x /opt/py2_legacy/libs/*.so

# 再来检查
[root@rocky9 ~]# find /opt/py2_legacy/usr/lib64/python2.7/lib-dynload/ -name "*.so" | xargs -I {} sh -c "LD_LIBRARY_PATH=/opt/py2_legacy/libs ldd {}" | grep "not found"

# 因为上一条命令输出为空，即正常，所以现在能查看python版本
[root@rocky9 ~]# LD_LIBRARY_PATH=/opt/py2_legacy/libs /opt/py2_legacy/usr/bin/python2.7 --version
Python 2.7.5            # 这一行是输出


# 编写标准化启动脚本 (The Wrapper)
不能指望客户每次都输入那一长串 LD_LIBRARY_PATH。你需要创建一个全局命令，让它像原生程序一样好用

[root@rocky9 ~]# vim /usr/local/bin/python27
#!/bin/bash
    # 强制指定旧版库路径
    export LD_LIBRARY_PATH="/opt/py2_legacy/libs:$LD_LIBRARY_PATH"
    # 执行隔离目录下的 python 2.7 并透传所有参数
    exec /opt/py2_legacy/usr/bin/python2.7 "$@"


[root@rocky9 ~]# chmod +x /usr/local/bin/python27
[root@rocky9 ~]# python
python     python27   python3    python3.9  

# 测试 SSL（旧版 Python 最容易在现代 OS 上挂掉的地方）
[root@rocky9 ~]# python27 -c "import ssl; print('SSL: OK')"
SSL: OK

# 测试加密库
[root@rocky9 ~]# python27 -c "import hashlib; print('Hashlib: OK')"
Hashlib: OK

# 测试数据库连接（如果有安装 MySQLdb 等模块）
[root@rocky9 ~]# python27 -c "import socket; print('Socket: OK')"
Socket: OK






# Pip迁移(可选)
如果应用需要安装新的第三方库，Python 2.7 的 pip 大多已经无法连接现代镜像站
建议方案：在脚本中把旧版 site-packages 也包含进来
操作：将 CentOS 7 的 /usr/lib64/python2.7/site-packages这个目录拷贝到 /opt/py2_legacy/usr/lib64/python2.7/ 目录下

# 在 CentOS 7 上打包
[root@localhost ~]# tar -czvf py27_site_packages.tar.gz /usr/lib64/python2.7/site-packages
[root@localhost ~]# scp py27_site_packages.tar.gz root@172.16.186.140:~                     # 发送到rocky9上


# 在 Rocky 9 上解压到对应目录
# 注意：确保路径 /opt/py2_legacy/usr/lib64/python2.7/ 已经存在，本次实验是有/opt/py2_legacy/usr/lib64/python2.7/site-packages/这个目录，且里面非空
因为在你最初解压 python27_base.tar.gz 时，这个目录作为 Python 标准安装结构的一部分已经被创建了
里面的内容通常是 Python 自带的一些基础工具(比如README、setuptools、pip的原始文件等)
[root@rocky9 ~]# tar -xzvf py27_site_packages.tar.gz -C /opt/py2_legacy/usr/lib64/python2.7/ --strip-components=4

# 扫描site-packages里的"隐藏依赖"
这些第三方库是“二次依赖”的高发区。请执行这个深度扫描：
[root@rocky9 ~]# find /opt/py2_legacy/usr/lib64/python2.7/site-packages/ -name "*.so" | xargs -I {} sh -c "LD_LIBRARY_PATH=/opt/py2_legacy/libs ldd {}" | grep "not found" | awk '{print $1}' | sort | uniq


# 跨越了最难的 SELinux 和 NSS 加密库 兼容性大山
[root@rocky9 ~]# python27 -c "import rpm; print('Success')"
Success

# 给所有 site-packages 下的扩展模块加上可执行权限
[root@rocky9 ~]# find /opt/py2_legacy/usr/lib64/python2.7/site-packages/ -name "*.so" -exec chmod +x {} +


# 检查当前环境加载的所有库中，是否同时出现了同一个库的两个大版本
[root@rocky9 ~]# LD_LIBRARY_PATH=/opt/py2_legacy/libs python27 -c "import rpm; import ssl; import socket; print('Link check passed')"
Link check passed              # 回显




# 用python27 your_script.py运行自己的脚本大概率会报错，原因在于之前的 import rpm 成功是建立在手动设置了环境变量的基础上的。如果直接运行，系统会找不到你搬运的那些旧版 .so 库和 site-packages 路径
[root@rocky9 ~]# python27 --version
Python 2.7.5

# 解决
[root@rocky9 ~]# cat /usr/local/bin/python27
#!/bin/bash
    # 强制指定旧版库路径
    export LD_LIBRARY_PATH="/opt/py2_legacy/libs:$LD_LIBRARY_PATH"
    export PYTHONPATH=/opt/py2_legacy/usr/lib64/python2.7/site-packages:/opt/py2_legacy/usr/lib/python2.7/site-packages
    # 执行隔离目录下的 python 2.7 并透传所有参数
    exec /opt/py2_legacy/usr/bin/python2.7 "$@"


[root@rocky9 ~]# python27 -c "import rpm; import ssl; print('Everything is OK')"
Everything is OK

[root@rocky9 ~]# python27 --version
Python 2.7.5

[root@rocky9 ~]# python27 -c "import urllib; print(urllib.urlopen('https://www.qq.com').getcode())"
200
