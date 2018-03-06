derivekey.py
===

derivekey.py文件用于生成随机数并写入secret.key文件

命令行参数：

- '--password': 根据参数创建rscoin.Key，并打印公钥的base64编码
- '--store'：调用urandom(32)生成32字节的随机字符串，写入secret.key文件，并输出公钥的base64编码
