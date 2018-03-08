__init__.py
===

__init__.py文件定义了Key和Tx两个类，分别用来操作密钥对和交易

# Key类

## 属性及构造方法

包含私钥，公钥，及optim属性

```python
_globalECG = EcGroup(713)

def __init__(self, key_bytes, public=True):
	self.G = _globalECG
	if public:
		self.sec = None
		self.pub = EcPt.from_binary(sha256(key_bytes).digest())
		self.optim = None
	else:
		self.sec = Bn.from_binary(sha_256(key_bytes).digest())
		self.pub = self.sec * self.G.generator()
		self.optim = do_ecdsa_setup(self.G, self.sec)
```

## 普通方法

### sign方法

用公私钥对签名一个32字节的消息

```python
def sign(self, message):
	assert len(message) == 32
	assert self.sec is not None
	r, s = do_ecdsa_sign(self.G, self.sec, message, self.optim)
	r0, s0 = r.binary(), s.binary()
	asser len(r0) <= 32 and len(s0) <= 32
	sig = pack("H32sH32s", len(r0), r0, len(s0), s0)
	return sig
```

### verify方法

使用公钥校验消息和签名

```python
def verify(self, message, sig):
	assert len(message) == 32
	lr, r, ls, s = unpack("H32sH32s", sig)
	sig = Bn.from_binary(r[:lr]), Bn.from_binary(s[:ls])
	return do_ecdsa_verify(self.G, self.pub, sig, message)
```

### id方法

返回公钥的指纹

```python
return sha256(self.pub.export()).digest()
```

### export方法

将公私钥用字符串的形式导出

```python
def export(self):
	sec = None
	if self.sec is not None:
		sec = self.sec.binary()
	return (self.pub.export(EcPt.POINT_CONVERSION_UNCOMPRESSED), sec)
```

# Tx类

## 属性及构造方法

包含InputTx、OutputTx、R、ser等属性

```python
def __init__(self, inTx=[], outTx=[], R=None):
	self.inTx = inTx
	self.outTx = outTx
	self.R = R
	if self.R is None:
		self.R = urandom(32)
	self.ser = None
```

## 普通方法

### serialize方法

将交易序列化为一个字符串

```python
def serialize(self):
	if self.ser is not None:
		return self.ser

	ser = pack("HH", len(self.inTx), len(self.outTx))
	ser += pack("32s", self.R)

	for intx in self.inTx:
		ser += pack("32sI", intx.tx_id, intx.pos)
	for outtx in self.outTx:
		ser += pack("32sQ", outtx.key_id, outtx.value)
	self.ser = ser
	return ser
```

### __eq__方法

比较两个Tx是否相同

```python
def __eq__(self, other):
	return isinstance(other, Tx) and self.id() == other.id()
```

### __ne__方法

比较两个Tx是否不同

```python
def __ne__(self, other):
	return not (self == other)
```

### parse方法

解析数据，即反序列化的过程

```python
def parse(data):
	idata = data

	i=0
	Lin, Lout, R = unpack("HH32s", data[i:i+4+32])
	i += 4+32

	inTx = []
	for _ in range(Lin):
		idx, posx = unpack("32sI", data[i:i+32+4])
		inTx += [InputTx(idx, posx)]
		i += 32+4
	outTx = []
	for _ in range(Lout):
		kidx, valx = unpack("32sQ", data[i:i+32+8])
		outTx += [OutputTx(kidx, valx)]
		i += 32+8
	tx = Tx(inTx, outTx, R=R)

	tx.ser = idata
	return tx
```

### id方法

返回tx的序列化的摘要

```python
return sha256(self.serialize()).digest()
```

### get_utxo_in_keys方法

为有效交易的utxo条目

```python
in_utxo = []
for intx in self.inTx:
	inkey = pack("32sI", intx.tx_id, intx.pos)
	in_utxo += [inkey]
return in_utxo
```

### get_utxo_out_entries方法

获取有效交易输出的utxo

```python
out_utxo = []
for pos, outtx in enumerate(self.outTx):
	out_utxo = []
	for pos, outtx in enumerate(self.outTx):
		outkey = pack("32sI", self.id(), pos)
		outvalue = pack("32sQ", outtx.key_id, outtx.value)
		out_utxo += [(outkey, outvalue)]
	return out_utxo
```

### check_transaction方法

根据给定的证据校验交易是由有效

如果是issue交易，校验inTx是否为空，keys和sigs的长度是否为1，然后交易签名。

如果是其他交易，调用Tx.parse方法解析交易，校验交易id是否匹配，调用check_transaction_utxo方法校验utxo。

### check_transaction_utxo方法

校验交易的UTXO

如果是issue交易，校验签名。

如果是其他交易，对于每个输出，交易交易id是否匹配，key是否匹配，签名是否匹配，价值是否匹配。