fabfile.py
===

fabfile.py文件定义了使用fabric部署系统的脚本。

# 机器相关配置

RSCoin系统的部署采用aws云平台的虚拟机，通过引入boto3模块进行调用。

```pyhton
import boto3
ec2 = boto3.resource('ec2')
```

## 获取机器

调用ec2.instance获取虚拟机实例并进行过滤，然后进行排序

```python
# 获取虚拟机名称
def get_aws_machines():
	instances = ec2.instances.filter(Filter[{'Name': 'instance-state-name', 'Values': ['running']}])
	return ['ubuntu@' + i.public_dns_name for i in instances] 

# 解析查找虚拟机名字, 无用
def parse_machines(s):
	urls = re.findall("ec2-.*.compute.amazonaws.com", s)
	names = [('ubuntu@' + u) for u in urls]
	return names

all_machines = sorted(get_aws_machines()) 
```

## 设置servers和clients

分配一半机器为servers，一半机器为clients

```python
servers = all_machines[:len(all_machines) / 2]
clients = all_machines[len(all_machines) / 2 :]

# 如果对server和client数量有限制，取前几个机器
...

# 设置角色

env.roledefs.update({
	'servers': dyn_server_role,
	'clients': dyn_client_role
})
```

## 虚拟机操作

### 启动

如果all_machines的数量不足60，调用ec2.create_instances启动缺少的机器

```python
@runs_once
def ec2start():
	if len(all_machines) < NUM_MACHINES:
		missing = NUM_MACHINES - len(all_machines)
		ec2.create_instances(
			ImageId='ami-178be960', 
            InstanceType='t2.micro',
            SecurityGroupIds= [ 'sg-ae5f0fcb' ],
            MinCount=missing, 
            MaxCount=missing )
```

### 打印虚拟机列表

调用ec2.instances.all()方法获取所有机器

```python
@runs_once
def ec2list():
	instances = ec2.instances.all()
	for instance in instances:
		print(instance.id, instance.state["Name"], instance.public_dns_name)
```

### 停止虚拟机

获取ec2实例并停止

```python
@runs_once
def ec2stop():
	instances = ec2.instances.filter(Filter=[{'Name': 'instance-state-name', 'Values': ['running']}])
	ids = [i.id for i in instances]
	try:
		ec2.instances.filter(InstanceIds=ids).stop()
		ec2.instances.filter(InstanceIds=ids).terminate()
	except Exception as e:
		print e
```

# 初始化

初始化过程首先调用derivekey.py文件中的功能产生一个password，存储到secret.key中。然后执行passcache操作，即安装sysbench、安装petlib、克隆rscoin项目

```python
@roles("servers", "clients")
@parallel
def passcache():
	sudo('rm -rf /home/ubuntu/projects/rscoin')
	sudo('apt-get install -y sysbench')
	with cd('/home/ubuntu/projects'):
		sudo('pip install petlib --update')
		run('git clone https://github.com/gdanezis/rscoin.git')
```

# 部署

deploy方法：依次执行gitall、keys、loaddir、loadsecret方法进行部署

## gitall方法

在所有servers和clients中执行git pull操作更新代码

## keys方法

在servers端进行。

首先加载secret.key文件中的密码到env["rsdir"]中
然后在服务端服务端执行derivekey.py中的store操作生成密码，然后将结果放在env["rsdir"]["directory"]中。

最后将env["rsdir"]数据dump到directory.conf文件中

## loaddir方法

将本地directory.conf文件传到**服务端**和**客户端**的directory.conf

## loadsecret方法

将本地的secret.key上传到客户端的clients

# 操作

## 启动start

在服务端执行twistd -y recserver.tac.py命令启动remitte后台任务

## 停止stop

在服务端和客户端杀掉twistd进程

## 清理clean

在服务端进行，删除experiment*目录和keys-*文件

# 性能测试

对服务器和本地机器性能进行测试，包括进行签名、哈希等操作的性能，CPU多线程基准测试，

## 服务器操作性能测试

执行tests/test_rscservive.py文件中的test_full_client方法和tests/test_Tx.py中的test_timing方法，计算签名速率、验签速率、哈希速率和检查交易速率。然后求平均值和标准差

```python
@roles("servers")
def time():
	with cd('/home/ubuntu/projects/rscoin'):
		x = run('py.test -s -k "full_client"') + "\n\n"
		x += run('py.test -s -k "timing"')

		for k, v in re.findall("(.*:) (.*) / sec", x):
			env.timings[k] += [float(v)]

		import numpy as np:
		f = file("remote_timings.txt", "w")
        for k, v in env.timings.iteritems():
            f.write("%s %2.4f %2.4f\n" % (k, np.mean(v), np.std(v)))
```

## 服务器多线程基准测试

采用sysbench工具对cpu的多线程性能进行基准测试

```python
@roles("servers")
@parallel
def cpu():
	out = run("sysbench --test=cpu --cpu-max-prime=2000 run")
	# 写入文件
	...
```

## 本地机器多线程基准测试

采用sysbench工具对本地机器多线程性能进行基准测试

# 实验

## 实验I

依次执行experiment1run、experiment1pre、experiment1actual、experiment1collect方法进行实验。

最后在本地执行exp1plot模块生成图表，执行estthroughput.py统计吞吐量

```python
execute("experiment1run")
execute("experiment1pre")
execute("experiment1actual")
execute("experiment1collect")

local("python exp1plot.py experiment1")
local("python estthroughput.py %s > %s/state.txt" % (env.expname, env.expname))
```

### expement1run方法

在客户端执行。首先执行“python simscript.py 2000 payments.txt”模拟生成100地址、2000个输出，并将[tx.serilized, pubkey, sig]放入payments.txt-issue文件中。创建1000个交易并将其放入payments.txt-r1文件中。创建1000个交易并将其放入payments.txt-r2文件中。

然后执行"./rsc.py --play payments.txt-issue > %s/issue-times.txt"发行货币。

### experiment1pre方法

执行"./rsc.py --play payments.txt-r1 --conn 30 > %s/r1-time.txt"模拟预提交交易

### experiment2pre方法

执行"./rsc.py --play payments.txt-r2 > %s/r2-time.txt"模拟真正执行交易

### experiment1collect方法

在客户端运行，将客户端的issue-time.txt, r1-time.txt, r2-time.txt放到拉到本地

## 实验II

即在本地执行实验I的内容，但发行数为1000

## 实验三

测试remitte的数量对系统影响的继承测试。

remitte数量为[10, 15, 20, 25, 30]下进行实验I的内容。