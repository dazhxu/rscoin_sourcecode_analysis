simscript.py
===

首先生成100个地址，地址的格式为(k.id(), k.pub.export(), k)，其中k是32字节的随机字符串生成的RSCoin.Key对象，k.pub.export()是k生成的公钥，k.id为k生成的公钥的SHA256哈希摘要。

生成给定参数个币地址。首先创建交易tx=Tx([], [OutputTx(key_id, 1)])，然后用key_special对tx.id()进行签名，模拟货币发行，用央行地址对输出进行签名。将[tx.serialize()， pub_special, sig]的base64编码写入文件payment.txt-issue文件。

创建转账交易，并预提交交易。首先将上一步生成的((tx.id(), 0, all_keys, 1), tx)写入active_addrs列表，并打乱顺序。从active_addrs中pop出两个tx，创建一个交易tx=Tx([InputTx(tx1_id, pos1), InputTx(tx2_id, pos2)], [OutputTx(k1id, 1), OutputTx(k2id, 1)]), 调用package_query方法查询交易，调用parse_Tx_bundle解析交易，调用tx.check_transaction方法检查交易。将其写入payment.txt-r1文件中。

创建转账交易并提交交易。与上一步步骤相同，写入payment.txt-r2文件中