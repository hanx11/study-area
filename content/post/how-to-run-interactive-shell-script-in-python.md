+++
title = 'How to Run Interactive Shell Script in Python'
date = 2023-09-27T11:48:54+08:00
draft = false
+++


最近有个功能需要使用`gpg`对服务器为不同工厂生成的OTP文件加密，且每个工厂的`gpg`密钥都不固定。

由于服务是运行在容器内的，所以需要在执行加密的前导入配置的`gpg`密钥，并将其改成可以可信任的。
`gpg`在导入密钥后，修改为可信任度时需要在指令执行过程中与`shell`有几次交互过程，所以简单的
`os.system(cmd)`无法完成操作。

通过一番资料查阅，最终找到了`pexpect`，可以完美的解决在执行`shell`命令的过程中需要与交互的问题。

示例代码如下：
```python 

import pexpect

# 导入工厂的gpg密钥
gpg_key_file = "F2A3D51D6164AA724461D02D26BF344D2FEC2866.key"
child = pexpect.spawn("gpg --import {}".format(gpg_key_file))
print(child.read())
child.close()

# 修改gpg密钥的信任等级
gpg_key_name = "F2A3D51D6164AA724461D02D26BF344D2FEC2866"
child = pexpect.spawn('gpg --edit-key {}'.format(gpg_key_name))
child.expect("gpg")
child.sendline('trust')
child.sendline('5')
child.sendline('y')
child.sendline("quit")
child.close()
```

#### 参考资料
- [GPG入门教程](https://www.ruanyifeng.com/blog/2013/07/gpg.html)
- [Pexpect Document](https://pexpect.readthedocs.io/en/stable/index.html)
