# Introduction #

Add your content here.


# Details #

9.  状态格式

> 每个用户有一个状态文件，文件名和用户名一样。缺省的状态文件路径/etc/otpstate。
> 格式：
> > ver:user:chal:csd:rd:failcount:authtime:mincardtime:

> 例子 (一些项可以为空):
> > 5:bob:12345678:::0:0:0:


> 最后的":"为必须。
<pre>
ver: 5<br>
user: 用户名<br>
chal: 前一个计数器的值 (challenge,用于基于事件的口令)<br>
csd: 口令特有的值（card-specific data）<br>
rd: rwindow (softfail) data<br>
failcount: 失败次数，如果让一次验证成功为0,如果不成功为连续失败次数<br>
authtime: 最后一次认证的时间 (成功或失败)<br>
mincardtime: 最后一次成功认证的时间<br>
</pre>
<pre>
challenge和csd项决定一个完整的状态(state)。其它项用来防止密码猜测，用户同期和空状态初始化。<br>
<br>
如果状态文件不存在，处理以口令而定。<br>
对于TRI-D,模块将初始化状态。没有必要自己生成初始数据。<br>
对于CRYPTOCard,用户必须非同期的进行验证以初始化状态。<br>
如果CRYPTOCard口令没有安装为支持非同期验证，或者在服务器的定义文件里设置为不允许，就需要手动生成的初始化状态文件。<br>
然后把状态文件的challenge项设为空。<br>
为了手动重设被锁定CryptoCard，把failcount设为0。如果用户偏离同期太远，也需要重设challenge项。<br>
以上关于CRYPTOCard的说明也适用于hotp和x9.9口令。<br>
<br>
注：authtime为Unix时间的16进制的值。<br>
类似以下命令输出的值。<br>
perl -e ' printf("%x\n",time()) '<br>
<br>
</pre>