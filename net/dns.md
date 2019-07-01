# dns
dns是让人使用名称, 让路由器使用 IP 地址进行通信的机制.

递归查询: dns server接收到客户机请求，**必须使用一个准确的查询结果回复客户机**. 如果dns server本地没有存储查询DNS 信息，那么该服务器会询问其他服务器，并将返回的查询结果提交给客户机. 主体是dns server.

迭代查询: DNS 服务器会向客户机提供其他能够解析查询请求的DNS 服务器地址，当客户机发送查询请求时，DNS 服务器并不直接回复查询结果，而是告诉客户机另一台DNS 服务器地址，客户机再向这台DNS 服务器提交请求，依次循环直到返回查询的结果为止. 主体是 客户机.

`/etc/resolv.conf`里放了支持的nameserver.
`dig www.baidu.com +trace`可了解整个dns查询过程.

> Let's Encrypt支持使用ACMEv2协议, 其可基于DNS(TXT DNS记录)来验证申请者对域名的所有权，从而签发证书.

### 多重响应
dns server返回多条dns记录, 起到负载均衡的效果.