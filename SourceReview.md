# Introduction #

Add your content here.


# Details #

Add your content here.  Format your content with:
  * Text in **bold** or _italic_
  * Headings, paragraphs, and lists
  * Automatic links to other wiki pages

1.Token的初始化需要两次连续输入在客服端生成的OTP
> －－usability需要改进

2.工作流程
```
 main(main.c) 启动daemon 
  利用xpthread_create启动accept_thread(accept_thread.c)
  {
    accept socket;
    利用xpthread_create启动work_thread(work_thread.c);
    {
     read socket。
     读取request.version(版本信息)。如果version = 2,读取信息至otp_request_t;如果version = 1,读取信息至otp_request_v1_t。      
     进行一些类型检查。
     调用在otp.c里定义的验证函数verify(config_t *config, const otp_request_t *request, otp_reply_t *reply);
     {
       根据配置文件otpd.conf的设置,调用file_get(userops/file.c)或者ldap_get(userops/ldap.c)
来取得用户信息。
     }
     write socket;
    }
  } 
```
failover\_thread.c
```
　从main.c启动的用来监视gsmd状态的Thread。
```
gsmd.c
```
  Global State Management Deamon
```
optauth.c
```
  利用socket通信，向daemon(OTPD)发送验证请求并接受OTPD的回复。
  验证请求用struct: otp_request_t
  回复用struct: otp_reply_t
```
main.c
```
 系统初始化
  config = config_init();  // /etc/otpd.confファイルを読み込んでconfigファイルを初期化
  cardops_init();    //初始化cardops全局变量         
  sig_init();        //signal的初始设定
  lock_init();       //初始化mutex
  daemonize();       //daemon初始化
  accept_thread的启动
  sigwait            //让main程序处于等待signal状态
```
state.c
```
  提供读写user的state文件用函数。利用lock.c里的函数来lock,unlock state文件。
  在/etc/otpd.conf文件里可以设置状态管理方式(local/gsmd)和状态文件路径(local方式时）  

  注:每个Token的状态以文件方式保存在服务器上。 
  # ls /etc/otpstate   
    Token001  Token002  Token003 
  # cat /etc/otpstate/Token001   
    5:Token001:0000000000000009:::0:4bea8f01:0:

  * 需要改进Token状态的存取方式。
    现在由于是拷贝一个临时文件，再重新生成状态文件，不管原先文件的permision是多少，新文件都会变成644。
```
lock.c
```
  用一个数组，里面包含64个链表来管理user的排他处理状态。利用username的hash值来决定
每个user应该在哪一个链表里管理。
  lock_get:从链表中取出指定用户的ulock_t
  lock_put:追加ulock_t到链表　　
```
otp.c
```
  系统最重要的文件，提供verify函数来验证Token。
```
xfunc.c
```
  一些系统函数的封装。例，
/* guaranteed setsid */
void
_xsetsid(const char *caller)
{
  if (setsid() == (pid_t) -1) {
    mlog(LOG_CRIT, "%s: setsid: %s", caller, strerror(errno));
    exit(1);
  }
}
```
userops/file.c,ldap.c
```
动态口令的shared secret可以保存在/etc/otppasswd文件上，也可以保存在LDAP服务器上。
通过/etc/otpd.conf配置文件可以更改设置。
file.c用于otppasswd文件的访问，ldap.c用于LDAP服务器的访问。
```
config.y
```
  main.c里使用到的config_init()函数在这个文件。
  config_init调用userotps_init/config.y,后者根据配置文件初始化userops来决定利用file.c
或者ldap.c。
```