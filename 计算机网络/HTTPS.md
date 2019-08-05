# HTTPS与HTTP

HTTP 有以下安全性问题：

- 使用明文进行通信，内容可能会被窃听；
- 不验证通信方的身份，通信方的身份有可能遭遇伪装；
- 无法证明报文的完整性，报文有可能遭篡改。

HTTPS 让 HTTP 先和 SSL（Secure Sockets Layer）通信，再由 SSL 和 TCP 通信，使用了隧道进行通信。通过使用 SSL，HTTPS 具有了加密（防窃听）、认证（防伪装）和完整性保护（防篡改）。

## 加密



## 参考资料

[http](https://github.com/CyC2018/CS-Notes/blob/master/notes/HTTP.md#%E4%B9%9Dget-%E5%92%8C-post-%E6%AF%94%E8%BE%83)

[**Https原理及流程**](https://www.jianshu.com/p/14cd2c9d2cd2)

