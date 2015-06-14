# Introduction #
系统将要追加的功能


# Details #
1. 生成一个GUI的管理界面，以便用来管理发布的动态口令
> 使用语言：PHP
> 主要功能：
  * 管理每个动态口令的状态，可以失效，暂时失效，回复口令
  * 每个公司的系统管理者进去后能浏览，查询该公司正在利用的所有的动态口令
  * report功能
2. API
  * 向Web服务器,Radius服务器等提供验证用API
> 目前的系统提供一个plugin，但用于这个plugin是用Unix domain socket来给动态口令服务器通信，它们必须在同一host上。
  * 上述管理功能也尽量提供API
3. 动态口令的初始化
  * 提供在客户端和服务器端进行共享动态口令的shared secret的方法
4. 在现有系统里面使用到了Helix加密方式，考虑到Helix已经不再被认为是一个安全，以后打算改成其它方式。
> > 关于Helix: http://en.wikipedia.org/wiki/Helix_%28cipher%29
5. 对基于时间的Token的支持