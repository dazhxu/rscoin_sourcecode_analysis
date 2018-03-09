rscservice.py
===

rscservice.py文件定义了若干数据处理方法和rsc服务

# 模块方法

## load_setup方法

从setup_data中加载配置

```python
def load_setup(setup_data):
	structure = loads(setup_data)
	structure["special"] = b64decode(structure["special"])
	structure["directory"] = [(b64decode(a), b, c) for a, b, c in structure["directory"]]
	return structure
```

## package_query方法

将交易打包。

参数：

- tx：交易
- tx_deps：交易依赖的其他交易
- keys：密钥

首先将tx序列化，然后将tx_deps中的交易序列化，用keys中的密钥对tx.id()签名。最后，对items进行base64编码，并取摘要

```python
def package_query(tx, tx_deps, keys):
	items = [tx.serialize()]
	for txi in tx_deps:
		items += [ txi.serialize() ]
	for k in keys:
		items += [ k.export()[0] ]
	for k in keys:
		items += [ k.sign(tx.id()) ]

	dataCore = map(b64encode, items)

	H = sha256(" ".join(dataCore)).digest()
	data = " ".join(["xQuery", str(len(dataCore))] + dataCore)
	return H, data, dataCore
```

## unpackage_query_response方法

解析query回应

```python
def unpaclage_query_response(response):
	resp = response.strip().split(" ")
	code = resp[0]
	if code == "OK" or code == "Pong":
		resp[1:] = map(b64decode, resp[1:])
	return resp
```

## package_commit方法

打包commit消息

## unpackage_commit_response方法

解析commit回应

## package_issue方法

打包issue交易

# RSCFactory类

继承twisted.internet.protocol.Factory类

## 属性与构造方法

参数：

- secret：密钥串
- directory：配置数据
- special_key：自身密钥
- conf_dir：配置目录
- N：

首先，根据参数初始化相应属性。

打开数据库

```python
self.dbname = 'key-%s' % hexlify(keyID)
self.logname = 'log-%s' % hexlify(keyID)
if conf_dir:
	self.dbname = join(conf_dir, self.dbname)
	sekf.logname = join(conf_dir, self.conf_dir)

if RSCFactory._sync:
	self.db = dbm.open(self.dbname, 'c')
	self.log = dbm.open(self.logname, 'c')
else:
	self.db = {}
	self.log = {}
```

##普通方法

### buildProtocol方法

创建一个RSCProtocol对象

### process_TxQuery方法

处理query请求，校验签名、校验输入地址是否在utxo中、交易输入是否被用过、将其添加到spent交易中并从utxo删除

参数：

- data：交易数据

首先获取utxo并检查至少有一个Input在这个server中

```python
mainTx, otherTx, keys, sigs = data
mid = mainTx.id()
inTxo = mainTx.get_utxo_in_keys()

should_handle_ik = []
for ik in inTxo:
	lst = self.get_authorities(ik)
	if self.key.id() in lst:
		should_handle_ik += [ ik ]
if should_handle_ik == []:
	return False
```

检查交易结构是否是正确的

```python
all_good = mainTx.check_transaction(otherTx, keys, sigs)
if not all_good:
	return False
```

检查所有的input是否都存在

```python
for ik in should_handle_ik:
	if ik in self.log and self.log[ik] == mid:
		continue
	elif ik not in self.db:
		return False
```

将所有input从utxo中删除

```python
for ik in should_handle_ik:
	if ik in self.db:
		del self.db[ik]
	self.log[ik] = mid
```

在签名之前进行保存

```python
if RSCFactory._sync:
	self.db.sync()
	self.log.sync()
```

### process_TxCommit方法

处理交易提交

首先检查这个交易是否被这个服务器处理

```python
H, mainTx, otherTx, keys, sigs, auth_pub, auth_sig = data
ik = mainTx.id()
lst = self.get_authorities(ik)
should_handle = (self.key.id() in lst)

if not should_handle:
	return False
```

检查所有的签名

```python
all_good = True
pub_set = []
for pub, sig in zip(auth_pub, auth_sig):
	key = rscoin.Key(pub)
	pub_set += [ key.id() ]
	all_good &= key.verify(H, sig)

if not all_good:
	return False
```

检查交易的格式

```python
all_good = mainTx.check_transaction(otherTx, keys, sigs, masterkey=self.special_key)
if not all_good:
	return False
```

检查是不是包含所有的验证者

```python
mid = mainTx.id()
inTxo = mainTx.get_utxo_in_keys()
for itx in inTxo:
	aut = set(self.get_authorities(itx))
	all_good &= (len(aut & pub_set) > len(aut) / 2)

if not all_good:
	return False
```

检查交易还没有被花费

```python
all_good &= (mid not in self.log)
if not all_good:
	return False
```

更新outTx条目并存储

```python
for k, v in mainTx.get_utxo_out_entries():
	self.db[k] = v
if RSCFactory._sync:
	self.db.sync()
```

### get_authorities方法

对于特定的xID返回其验证者的key

调用模块方法get_authoritis

```python
d = sorted(directory)

if len(d) <= N:
	auths = [di[0] for di in d]
else:
	i = unpack("I", xID[:4])[0] % len(d)
	auths = [d[(i + j -1) %len(d)][0] for j in range(N)]
return auths
```

# RSCProtocol类

继承twisted.protocols.basic.LineReceiver类

## 属性与构造方法

接受一个RSCProtocol类

```python
def __init__(self, factory):
	self.factory = factory
```

## 普通方法

### lineReceived方法

一个简单的de-multiplexer，接受一个消息line

如果是xQuery消息，调用handle_Query方法获取签名；
如果是xCommit消息，调用handle_Commit方法处理交易；
如果是Ping消息，发送一个Pong消息。

### return_Err方法

发送一个Error消息

### handle_Query方法

首先，调用parse_Tx_bundle解析数据

然后调用self.factory.process_TxQuery方法检查交易。如果检查通过，返回对头的签名。

### handle_Commit方法

首先，调用parse_Tx_bundle解析数据，然后调用self.factory.process_TxCommit方法，返回交易id的签名。

### parse_Tx_bundle方法

解析数据包