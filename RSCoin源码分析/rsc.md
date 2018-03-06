rsc.py
===

rsc模块提供命令行入口

# main方法

提供命令行解析入口及相对应的处理流程。

首先加载directory.conf文件，从中读取(kid, ip, port)信息，及special信息。

然后解析命令行参数

- "--dir": 列出所有的mintette
- "--mock": 不会连接到网络
- "--balances": 列出所有地址的余额
- "--issue": 向一个地址发行一定数量的货币，后有两个参数"value"和"address"
- "--pay": 一个地址向另一个地址转账一定数量的货币。后有三个参数"value", "dest_address"和"send_address"。
- "--newaddress": 给定一个具体的名字创建一个地址。后有一个参数"name"
- "--storeaddress": 加载名字和key_id。后有两个参数"name"和"key_id"
- "--listaddress": 列出所有已知的地址
- "--play": 执行一组交易。后有一个参数"file"
- "--conn"：连接数，默认为20

最后根据参数执行相应处理

## dir参数

打印directory的(kid, ip, port)

## balances参数

首先加载key，然后创建ActiveTx对象，最后打印active.balances

```python
keys = load_keys()
active = ActiveTx("activetx.log", keys)
for (k, v) in active.balances().iteritems():
	print "%s\t%s RSC" % (k, v)
```

## listaddress参数

首先调用load_leys()方法加载key。然后打印地址和名字

```python
keys = load_keys()
for k in keys:
	if k[0] == "#":
		print "%s\t%s (%s)" %(k, keys[k][2], keys[k][1])
```

## newaddress参数

生成一个32字节的key，写入keychain.txt文件中

```python
sec_str = urandom(32)
k_sec = rscoin.Key(sec_str, public=False)
k_pub = k_sec.pub.export()
k_id = k_sec.id()

f = file("keychain.txt", "a")
data = "#%s sec %s %s" % (args.newaddress[0], b64encode(k_id), b64encode(sec_str))
f.write(date+"\n")
f.close()
```

## storeaddress参数

将输入的name和key写入keychain.txt文件中

## issue参数

首先，从secret.key中读取私钥，然后创建rscoin.Key对象。

然后调用load_keys方法加载keychain.txt文件，读取参数key_name对应的key_id。创建一个rscoin.Tx实例，并用secretkey对其进行签名。

```python
tx = rscoin.Tx([], [rscoin.OutputTx(key_id, int(value_str))])
sig = mykey.sign(tx.id())
```

将(tx, [mykey, sig])打包。如果参数有mock参数，直接打印数据包；否则，执行issue流程：

- 首先，获取认证器及相关的mintette信息。将数据广播到mintette

```python
auths = set(get_authorities(directory, tx.id()))
small_dir = [(kid, ip, port) for (kid, ip, port) in directory if kid in auths]

d = broadcast(small_dir, data)
```

- 创建回调函数，处理广播结果

```python
def r_process(results):
	for msg in results:
		parsed = unpackage_commit_response(msg)
		if parsed[0] != "OK":
			raise Exception("Response not OK.")
		pub, sig = parsed[1:]
		kx = rscoin.Key(pub)
		if not (kx.verify(tx.id(), sig) and kx.id() in auths):
			raise Exception("Invalid Signature.")
		auths.remove(kx.id())
	active = ActiveTx("activetx.log", keys)
	active.add(tx_ser)
	active.save(reactor)
	print " ".join(core)
```

- 添加回调函数

```python
d.addCallback(r_process)
d.addBoth(r_stop)
reactor.run()
```

## pay参数

首先，获取keys及发送者接受者的地址。根据keys创建ActiveTx对象，并从中查询大于val数量的交易(xval, txs)。

创建交易。首先构建val数量的输出，及xval-val的输出；然后从txs中查询并创建输入；最后创建交易。

```python
inTx_list = []
keys_list = []
for (tx_id, i, key_id, value) in txs:
	inTx_list += [ rscoin.Tx.parse(active.Tx[(tx_id, i, key_id, value)]) ]
	keys_list += [ rscoin.Key(b64decode(keys[key_id][3]), False) ]
	inTx += [ rscoin.InputTx(tx_id, i) ]
newtx = rscoin.Tx(inTx, outTx)
newtx_ser = newtx.serialize()
```

将交易添加到active中，并调用play方法执行交易。

```python
active.add(newtx_ser)
for k in txs:
	active.remove(k)
active.save(reactor)

sechash, query_string, core = package_query(newtx, inTx_list, keys_list)
d = play(core, directory)
d.addBoth(r_stop)
reactor.run()
```

## play参数

首先，从参数指定的文件中读取数据；

然后创建线程处理函数play_another_song

```python
def play_another_song(var):
	if var is not None and (not isinstance(var, float) or not isinstance(var, float)): 
		print "ERROR", var
	if cores != []:
		c = cores.pop()
		d = play(c, directory)
		d.addCallback(play_another_song)

		def replay():
			cores += [ c ]

		d.addErrback(replay)
		d.addErrback(play_another_song)
	else:
		threads.pop()
		if threads == []:
			reactor.stop()
```

创建conn参数个threads，调用play_another_song().

统计时间。

```python
t0 = default_timer()
reactor.run()
t1 = default_timer()
```

# 辅助方法

## load_keys方法

从keychain.txt文件中读取数据

## r_stop方法

停止reactor

```python
if results is not None and isinstance(results, Exception):
	print "Error", results
reactor.stop()
```

## broadcast方法

