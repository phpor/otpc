# Introduction #


# Details #
为了利用LDAP,需要修改配置文件/etc/otpd.conf,首先指定后台支持ldap
```
    # The backend to use to obtain per-user token data, file or ldap.
    # The default is file.
    #backend = file
    backend = ldap
```
然后在ldap节里面设置LDAP服务器访问信息。需要根据实际情况调整相应配置。
```
ldap {
        # The directory server/port.
        # The default host is localhost.
        # The default port is obtained using getservbyname("ldap", "tcp")
        # (which uses nsswitch to pick a source, but generally this is
        # /etc/services), falling back to a default of 389.
        host = localhost
        port = 389            

        # The user/pass to bind as.  The default is anonymous.
        binddn = dc=sichuan,dc=bazhong
        bindpw = secret

        # Where in the DIT to find user information (with token data).
        # No default. (Eg. ou=People,dc=example,dc=com)
        basedn = dc=sichuan,dc=bazhong
        # The search filter. A single %u will be substituted with the username.
        # No default. (Eg. uid=%u)
        filter = uid=%u    
        # 中略
    } # ldap
```
有关OpenLDAP的简单的安装方法，可以参考http://my.oschina.net/sec/blog/9506

向LDAP服务器里追加口令信息：
sample.ldif
```
dn: cn=Token1219,dc=sichuan,dc=bazhong
objectclass: person
objectclass: otpToken
sn: Token1219
cn: Token1219
otpVendor: hotp
otpModel: d6
otpKey1: 6a7af753cf5bfed9fa933169a19166fce8ea80ee
```

生成状态文件：
```
resynctool -1 155833 -2 578300 -u Token1219 -k 6a7af753cf5bfed9fa933169a19166fce8ea80ee > /etc/otpstate/Token1219
```

当运用otpauth来验证口令的时候，otpd就会从LDAP服务器取得shared secret来进行处理。
```
otpauth -u Token1219 -p 703087 -s /var/run/otpd/socket
```