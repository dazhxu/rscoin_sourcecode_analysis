rscserver.tac.py
===

使用twisted启动后台服务进程的脚本。

首先从secret.key文件中读出随机数并打印公钥。然后从directory.conf文件中读出系统配置并加装。然后启动应用及TCPserver服务

```python
application = service.Application("rscoin")
echoService = internet.TCPServer(8080, RSCFactory(secret, directory["directory"], directory["special"], N=3))
echoService.setServiceParent(application)
```