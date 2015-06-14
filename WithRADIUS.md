# Introduction #
测试和Apache,FreeRADIUS系统一起使用动态口令系统

# Details #
<pre>
概要：<br>
在这里介绍一起使用Apache,FreeRADIUS,OTPD系统的方法。系统的流程如下述：<br>
1.在访问需要验证的Apache服务器的时候，用户输入用户名和在动态口令<br>
令牌上生成的一次性密码<br>
2.Apache利用mod_auth_radius包，把用户名和密码传给FreeRADIUS<br>
3.FreeRADIUS利用rlm_otp包，把用户名和密码传给OTP服务器来验证<br>
4.OTP验证密码并把结果返回FreeRADIUS,FreeRADIUS把结果返回到Apache<br>
就这样实现在Apache服务器上的动态口令的利用<br>
环境：<br>
CentOS 5.5<br>
OpenSSL 0.9.8e<br>
Apache 2<br>
</pre>

## 第一步：安装FreeRADIUS ##
搜索可以利用FreeRADIUS的包：
```
# yum search freeradius2
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * addons: ftp.jaist.ac.jp
 * base: ftp.jaist.ac.jp
 * extras: ftp.jaist.ac.jp
 * updates: ftp.jaist.ac.jp
============================= Matched: freeradius2 =============================
freeradius2.i386 : High-performance and highly configurable free RADIUS server
freeradius2-krb5.i386 : Kerberos 5 support for freeradius
freeradius2-ldap.i386 : LDAP support for freeradius
freeradius2-mysql.i386 : MySQL support for freeradius
freeradius2-perl.i386 : Perl support for freeradius
freeradius2-postgresql.i386 : Postgresql support for freeradius
freeradius2-python.i386 : Python support for freeradius
freeradius2-unixODBC.i386 : Unix ODBC support for freeradius
freeradius2-utils.i386 : FreeRADIUS utilities
```
安装你需要的FreeRADIUS包：
```
# yum install freeradius2 freeradius2-krb5 freeradius2-ldap freeradius2-utils
```
安装成功的时候的显示信息的一部分
```
Installed:
  freeradius2.i386 0:2.1.7-7.el5         freeradius2-krb5.i386 0:2.1.7-7.el5    
  freeradius2-ldap.i386 0:2.1.7-7.el5    freeradius2-utils.i386 0:2.1.7-7.el5   

Complete!
```

试着在调试模式下启动RADIUS服务
```
radiusd -X
```
仔细阅读输出的信息，如果没有错误，可以发现很多有关RADIUS的信息。比如
它生成用于EAP-TLS的RADIUS服务器电子证书 /etc/raddb/certs/server.pem

在/etc/raddb/users文件里插入一行来测试RADIUS
> myradiususer Cleartext-Password := "password"

重新在调试模式下启动RADIUS服务
```
radiusd -X
```
利用radtest向RADIUS服务器发出验证请求，如果验证成功将显示下面内容。
```
# radtest myradiususer password 127.0.0.1 0 testing123
Sending Access-Request of id 130 to 127.0.0.1 port 1812
	User-Name = "myradiususer"
	User-Password = "password"
	NAS-IP-Address = 127.0.0.1
	NAS-Port = 0
rad_recv: Access-Accept packet from host 127.0.0.1 port 1812, id=130, length=20
```

## 第二步：安装Apache插件mod\_auth\_radius ##
这里利用apxs来给Apache安装扩展插件，首先安装apxs
```
# yum install httpd-devel.i386
```

下载mod\_auth\_radius的代码并安装
```
# wget ftp://ftp.freeradius.org/pub/radius/mod_auth_radius-1.5.8.tar
# tar xvf mod_auth_radius-1.5.8.tar
# cd mod_auth_radius-1.5.8
# apxs -i -a -c mod_auth_radius-2.0.c
```
注意：在mod\_auth\_radius-1.5.8.tar里面同时包含支持Apache不同版本的代码
<pre>
mod_auth_radius.c  用于 Apache 1.3.x<br>
mod_auth_radius-2.0.c  用于 Apache 2<br>
</pre>
试设置而定，在执行apxs之前，有需要运行下一行以便把openssl的头文件包含进去的时候
```
# C_INCLUDE_PATH=/usr/local/ssl/include;export C_INCLUDE_PATH
```
<pre>
==第三步：设置Apache以利用mod_auth_radius==<br>
参照FreeRADIUS包里面的httpd.conf文件，修改Apache的httpd.conf文件<br>
在其它的LoadModule行后面追加mod_auth_radius的指令<br>
</pre>
```
#
# Tell Apache to load the module.
#
LoadModule radius_auth_module /usr/lib/httpd/modules/mod_auth_radius-2.0.so
```
在其它 `<`IfModule`>` `<`/IfModule`>`后面追加对mod\_auth\_radius的设置信息
```
# Add the general configuration of mod_auth_radius
# either to the BOTTOM of httpd.conf
# or into <VirtualHost> configuration before per-directory settings
#
<IfModule mod_auth_radius-2.0.c>

#
# AddRadiusAuth server[:port] <shared-secret> [ timeout [ : retries ]]
#

# Use localhost, the standard RADIUS port, secret 'testing123',
# time out after 5 seconds, and retry 3 times.
AddRadiusAuth localhost:1812 testing123 5:3

#
# AuthRadiusBindAddress <hostname/ip-address>
#
# Bind client (local) socket to this local IP address.
# The server will then see RADIUS client requests will come from
# the given IP address.
#
# By default, the module does not bind to any particular address,
# and the operating system chooses the address to use.
#

#
# AddRadiusCookieValid <minutes-for-which-cookie-is-valid>
#
# the special value of 0 (zero) means the cookie is valid forever.
#
AddRadiusCookieValid 5
</IfModule>
```
省略掉注释,相当于在Apache的httpd.conf文件，使用到了两个指令
```
# 指定Radius服务器的地址，端口，共享的密码，失效时间和重试的次数
# AddRadiusAuth server:port shared-secret timeout:retries
AddRadiusAuth localhost:1812 testing123 5:3

# 指定cookie的有效时间
# AddRadiusCookieValid <minutes-for-which-cookie-is-valid> 
AddRadiusCookieValid 5
```

对要需要密码的文件夹进行访问控制的设置
```
#  A sample per-directory access-control configuration.

<Location /secure/>
# Tell the user the realm to which they're authenticating.
# This string should be configured for your site.
AuthName "Test radius atuthentication"

# Apache 2.x specific setting:
# Set RADIUS to be the provider for this basic authentication
AuthBasicProvider radius

# Make a local variation of AddRadiusCookieValid.  The server will choose
# the MINIMUM of the two values.
AuthRadiusCookieValid 5

# Set the use of RADIUS authentication at this <Location>"
AuthRadiusActive On

# require that mod_auth_radius return a valid user, otherwise
# access is denied.
require valid-user
</Location>
```
为了让文面显得简单，这里省掉了很多注释。如果省略掉所有的注释，实际上只是简单的加了加行指令，以要求访问 secure路径下的资源的时候要提供用户名和密码。
```
<Location /secure/>
AuthName "Test radius atuthentication"
AuthBasicProvider radius
AuthRadiusCookieValid 5
AuthRadiusActive On
require valid-user
</Location>
```
保存修改过的Apache的httpd.conf文件，重起Apache并启动Radius,这样当你从浏览器上访问
/secure下面的文件的时候，一个要求用户名的对话框将被弹出，参照Radius的users文件的
内容输入用户名和密码。
> 比如在/etc/raddb/users文件里事先插入了下面一行
> > myradiususer Cleartext-Password := "password"
这样在访问 /secure/testOTP.html的时候，就输入用户名：myradiususer,密码：password

## 第四步：设置使用动态口令 ##
在FreeRADIUS的包里面，有一个叫rlm\_otp的包，利用它可以实现FreeRADIUS和OTP服务器的连接。
编辑/etc/raddb/sites-available/default文件，在authenticate节里面追加文字"otp"。
例：
```
#  Please do not put "unlang" configurations into the "authenticate"
#  section.  Put them in the "post-auth" section instead.  That's what
#  the post-auth section is for.
#
authenticate {
        #
        #  Allow OTP authentication
        otp
```
在同一文件的authorize节里面追加文字"otp"。
```
#  Make *sure* that 'preprocess' comes before any realm if you
#  need to setup hints for the remote radius server
authorize {
        #
        #  Allow OTP authorization
        otp
```
在调试模式下启动FreeRADIUS(radiusd -X),如果rlm\_otp包装载成功，将能看到下面的
信息
```
 Module: Checking authenticate {...} for more modules to load
 Module: Linked to module rlm_otp
 Module: Instantiating otp
  otp {
	otpd_rp = "/var/run/otpd/socket"
	challenge_prompt = "Challenge: %s  Response: "
	challenge_length = 6
	challenge_delay = 30
	allow_sync = yes
	allow_async = no
	mschapv2_mppe = 2
	mschapv2_mppe_bits = 2
	mschap_mppe = 2
	mschap_mppe_bits = 2
  }
```

由于 /var/run/otpd/socket的权限问题，为了测试，这里改变该文件的访问权限
```
# chmod 777 /var/run/otpd/socket
```
再次访问 /secure/testOTP.html的时候，就可以输入动态口令来进行登录了。
注意：如果还没有建立起动态口令系统，请参照http://code.google.com/p/otpc/w/edit/Start