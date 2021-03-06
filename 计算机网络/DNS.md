# DNS

## 定义

DNS域名解析，将域名翻译成IP地址

## DNS解析流程

1.先去浏览器dns缓存里查找，如果没有就去本地DNS服务器找当前网站的映射；

2.如果没有调用操作系统的gethostbyname()函数，通过网卡向DNS根域名服务器发起UDP请求；

3.以www.taobao.com为例,解析的域名为www.taobao.com.，.是根域名，.com是顶级域名,.taobao是次级域名，www是三级域名。解析过程分级查询，如果本服务器没有，会返回下一个域的ip地址递归查询；

4.查到DNS服务器的ip地址返回。

## DNS在域名解析中用到的TCP协议

DNS规范了两种服务器，一种叫主服务器，一种是辅助服务器。主DNS服务器从本机读取DNS数据信息，发送给辅助服务器。当一个辅助服务器启动时，会与主DNS服务器通信，进行区域传送。

### 为什么域名解析用UDP

UPD快，但传输内容不能超过512字节，DNS的域名请求和响应一般都不会超过512字节；

区域传送使用TCP是因为TCP可靠性好，同时，DNS服务器上的内容可能大于512字节。

## CNAME

别名记录，也叫做规范名字，换句话说，就是把一个域名解析为一个域名，减小服务器更换后的域名 ip映射修改工作量。

由于CNAME的参与，域名解析的结果不再是一个IP地址，而是对应的CNAME。浏览器对CNAME进行解析，得到IP地址。在此过程中，CDN会根据用户地理位置，解析对应的IP地址，就近访问。配合负载均衡，提高访问速度。

### 域名解析选 A 记录还是 CNAME 记录

如果是长期建站、项目运营的话，一般都建议使用 CNAME 记录。CNAME 记录可用于 CDN 加速，通过 CDN 加速别名解析网站域名，这样既可以起到加速网站的作用，又能隐藏网站的真实 IP，减少被攻击的几率。现在的云服务器一般都接入了 BGP 多线路，至少是电信、联通、移动三线路，在更换 IP 的时候 CNAME 记录变，特别方便。

## 参考资料

[关于DNS不得不说的一些事](https://www.cnblogs.com/rjzheng/p/11395695.html)

[**域名 A 记录和 CNAME 记录区别在哪？如何选择？**](https://cloud.tencent.com/developer/article/1349559)

[**关于CNAME**](https://www.jianshu.com/p/65757b5c0762)

