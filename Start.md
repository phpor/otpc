怎样运行，测试otpc系统

# Introduction #
如果你已经有Linux的环境，请忽略下面有些步骤。在这里将介绍从Windows XP
下开始利用免费软件从零开始建立测试环境的方法。
环境：
<pre>
VWPlayer 3.0<br>
CentOS 5.0<br>
gcc<br>
OpenSSL<br>
</pre>

# Details #
下载并安装VWPlayer 3.0
> http://downloads.vmware.com/d/info/desktop_downloads/vmware_player/3_0

下载CentOS的image
> http://www.vmware.com/appliances/directory/820

运行VWPlayer并启动CentOS

运行以下命令安装gcc到CentOS
```
# yum install gcc gcc-c++ autoconf automake
```

下载安装openssl到CentOS
```
# cd openssl-*.*.*
# ./config shared
# make
# make test
# make install
```

下载otpd-3.1.0.tar.gz到CentOS
例: wget http://otpc.googlecode.com/files/otpd-3.1.0.tar.gz

# openssl sha1 otpd-3.1.0.tar.gz 的结果：21a3d4c6aa9081f21f2b32c8e16a013cd98c8abb
如果HASH值不一致，请检查下载的文件是否损坏。

以下命令安装现有的OTPD系统
```
# gtar zxvf otpd-3.1.0.tar.gz
# cd otpd-3.1.0/
# ./configure
# make
# make install
# mkdir /etc/otpstate
# touch /etc/otppasswd
# chmod 600 /etc/otppasswd
# chmod 700 /etc/otpstate
# mkdir /var/run/otpd
# touch /var/run/otpd/socket
```

利用适当的OATH对应的Token客户端,比如一个在iphone上的令牌口令软件：
<pre>
http://code.google.com/p/oathtoken/<br>
</pre>
另一个用perl写的OATH对应的Token：
<pre>
http://cpansearch.perl.org/src/IWADE/Authen-HOTP-0.02/<br>
</pre>

设置OTP服务器
把Token客户端的key共享给OTP服务器
```
# cat >> /etc/otppasswd
 Token001:hotp-d6:80d4d335194ae378fa901767596969bc213e6535
```
在这里，Token001是用户名，你可以根据需要做修改。
hotp-d6表示算法是htop,会生成6位的OTP
后面部分是Token客户端的key(每个客户端都会有一个唯一的秘密的key,由于这个系统和客户端程序没有有效的交换key的办法，为了测试这里只能直接拷贝)

在Token客户端生成两次OTP，用以下命令来同步客户端和OTP服务器
```
# resynctool -1 152979 -2 758041 -u Token001 -k 80d4d335194ae378fa901767596969bc213e6535 > /etc/otpstate/Token001
```

为了测试动态口令，首先启动OTP服务器
```
# otpd -D
  otpd: otpd 3.1.0 starting
  otpd: accept_thread: tid=3086179248
```

在Token客户端生成新的动态口令，用以下命令来验证它是否有效
```
# otpauth -u Token001 -p 995880 -s /var/run/otpd/socket
  0 (ok)
```
`*` 这里可能需要sshd或者telnet来访问CentOS。

如果出现 0 (ok)的话，说明动态口令通过了验证。由于由于resynctool的bug,第一次验证的时候将出现验证失败的错误。第二次以后将没有问题。 ＊1
```
# otpauth -u Token001 -p 005691 -s /var/run/otpd/socket
3 (authentication error)
```

每个Token的状态以文件方式保存在服务器上。
```
# ls /etc/otpstate
  Token001  Token002  Token003
# cat /etc/otpstate/Token001
  5:Token001:0000000000000009:::0:4bea8f01:0:
```
在/etc/otpstate文件夹里，状态文件以Token名作为文件名，在文件里面
0000000000000009 表示现在Token的使用次数。4bea9f01为一个基于Token的使用次数的值。

安装过程中的可能出现的问题
问题：运行.configure的时候，出现一下错误：
```
 checking openssl/md4.h usability... no
 checking openssl/md4.h presence... no
 checking for openssl/md4.h... no
 configure: error: unable to find openssl headers
```
解决办法：
> 在运行之前，将openssl/md4.h头文件所在路径追加到C\_INCLUDE\_PATH环境变量里面
> 例：
```
   #C_INCLUDE_PATH=/usr/local/ssl/include;export C_INCLUDE_PATH
```
在这里假设/usr/local/ssl/include/openssl/md4.h文件存在。请根据自身的环境设置路径

问题：运行.configure的时候，出现一下错误：
```
  configure: error: OpenSSL(libcrypto) is required
```
解决办法：
> 运行.configure的时候用with-openssl指定libcrypto包的位置
```
   #./configure --with-openssl=/usr/local/ssl
```
在这里假设/usr/local/ssl/lib/libcrypto.so文件存在。请根据自身的环境设置路径

问题：运行resynctool的时候，出现一下错误：
```
resynctool: error while loading shared libraries: libcrypto.so.0.9.8: cannot open shared object file: No such file or directory
```
解决办法：
在运行之前,把相关路径追加到LD\_LIBRARY\_PATH环境变量里面
```
 LD_LIBRARY_PATH=/usr/local/ssl/lib;export LD_LIBRARY_PATH
```

另一次成功运行了的环境：
<pre>
Parallels Desktop 5 for Mac<br>
CentOS 5.5<br>
gcc 4.1.2<br>
OpenSSL 0.9.8k<br>
</pre>

＊1 ： 使用最初版本的resynctool的时候，/etc/otpstate/Token名文件里面保存的次数为下一次的count数，而系统认为是当前count数，所以出现错误。
```
 例：
 5:MyToken:0000000000000003:::0:0:0:
为了和系统正确同步，应该为
 5:MyToken:0000000000000002:::0:0:0:
```
