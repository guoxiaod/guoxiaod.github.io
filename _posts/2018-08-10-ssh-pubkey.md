---
toc: true
date: 2018-08-10 15:16:00 +08:00
---
## SSH 公钥认证的配置方法 <sub>{{ page.date | date: '%Y-%m-%d' }}</sub>

### 概述

ssh 有很多种的认证方式，我们常用的就是密码和公钥方式了，因为公钥方式可以免去每次输入密码的限制，所以配置好了，可以节约好多时间

### 环境

假如我们有：

1. 192.168.1.101 (我们在这台服务器上生成公钥)
2. 192.168.1.102 (我们要从 192.168.1.101 登录到这台服务器上)

假如我们已经在 192.168.1.101 和 192.168.1.102 上创建了 bob 账号

假如我们选择使用 ecdsa 算法(目前 ssh 支持 dsa,ecdsa,ed25519,rsa,rsa1)

### 操作步骤

```bash
# 1. 登录 192.168.1.101, 输入用户名和密码
$ ssh 192.168.1.101 -l bob

# 2. 运行 ssh-keygen 生成公钥/私钥对
$ ssh-keygen -t ecdsa -N '' 
$ ls .ssh
id_ecdsa  id_ecdsa.pub

# 3. 把公钥分发到 192.168.1.102 服务器上
$ ssh-copy-id 192.168.1.102

# 4. 测试从 192.168.1.101 登录 192.168.1.102 是否需要密码
$ ssh 192.168.1.102
$ ls .ssh
authorized_keys

# 5. 测试从 192.168.1.102 登录 192.168.1.101 是否需要密码
$ ssh 192.168.1.101
# 这儿应该会提示你需要输入密码了, 因为在 192.168.1.102 上没有私钥 (id_ecdsa)

# 6. 从 192.168.1.101 上 scp 公钥/私钥 到 192.168.1.102 上
$ scp .ssh/{id_ecdsa.pub,id_ecdsa} 192.168.1.102:.ssh/

# 7. 再次从 192.168.1.102 上 登录 192.168.1.101
$ ssh 192.168.1.101
# 应该还是不行的，这儿缺少 authorized_keys 文件, 因为默认 sshd (ssh server) 是从 这个文件读取公钥的

# 8. 可以使用 ssh-copy-id 来生成这个文件，或者直接拷贝 id_ecdsa.pub 到 authorized_keys 即可
$ ssh-copy-id localhost  # 相当于将公钥分发到本地
# 或者
$ cp .ssh/id_ecdsa.pub .ssh/authorized_keys

```

### .ssh 目录下的文件

1. id_ecdsa
  - 私钥
  - 根据选择的算法不同，文件名也不同，格式为 'id_' 后面加上算法名称
  - rsa1 算法文件名则为 identity
1. id_ecdsa.pub
  - id_ecdsa 对应的 公钥
  - 文件名格式为 私钥文件名后面加上 '.pub'
  - ssh-copy-id 分发的就是这个公钥 
1. authorized_keys
  - 从其他服务器上分发过来的公钥会保存到这个文件中
  - 每行一个, 每行的数据会和原公钥文件 (id_ecdsa) 一致
1. known_hosts
  - 本机已经接受的, 可以直接登录到的其他服务器的公钥
  - 当我们第一次登录其他服务器的时候，提醒我们是否继续的就是确认是否接受要登录的服务器的公钥
  - 需要和上面的 id_ecdsa.pub 区分开
  - id_ecdsa.pub 是每个用户自己配置的，每个用户都不同
  - 而这儿的公钥是每台服务器一个, 安装服务器的时候生成的


### id_ecdsa


```bash
# 可以通过如下的命令来查看 id_ecdsa 文件的数据
# 对应到 rsa / dsa 算法，使用 rsa / dsa 替换下面命令中的 ec 即可(其他的没有找到命令)
$ openssl ec -in id_ecdsa  -text

# 可以通过如下的命令导出私钥对应的公钥
$ openssl ec -in id_ecdsa -pubout -out out.pub
```

### id_ecdsa.pub

```bash
# 可以通过如下的两个命令查看，out.pub 和 id_ecdsa.pub 中后面部分的数据是一致的 
$ openssl ec -in out.pub  -text -pubin
$ awk '{print $2;}' id_ecdsa.pub |base64 -d |hexdump  -e '/1 "%02x:"'; echo
```

### authorized_keys

```bash
# 每行一个保存了从其他服务器上分发过来的公钥
```

### known_hosts

```bash
# 这个文件的内容来源于其他的服务器的 /etc/ssh/ssh_host_{dsa,rsa,ecdsa,ed25519}_key.pub
# 上面的文件是通过下面的命令生成的(不要轻易执行哦, 会覆盖原先的文件)
$ ssh-keygen -A

# 上面的命令会同时生成所有支持的私钥/公钥
# CentOS 7.x 目前是通过服务 sshd-keygen.service 来自动生成这些文件
# 这个服务通过单次运行 /usr/sbin/sshd-keygen 这个命令来生成这些文件
$ /usr/sbin/sshd-keygen 

# 可以通过如下的命令查看某台服务器的公钥是否已经在这个文件中了
$ ssh-keygen -F github.com
# 可以通过如下的命令查看某台服务器的公钥的 fingerprint
$ ssh-keygen -F github.com -l -E {md5 | sha256}

# 如果某一台服务器重装了，则其公钥/私钥都会重新生成，known_hosts 保存的就需要更新
# 可以通过如下命令删除，然后 ssh 登录的登录就会重新获取了
$ ssh-keygen -R github.com

# 主动获取某一台服务器的公钥
$ ssh-keyscan -t {rsa | dsa | ecdsa | ed25519} github.com
$ ssh-keyscan -t {rsa | dsa | ecdsa | ed25519} github.com >> ~/.ssh/known_hosts
# 生成的数据每行三个字段，第一个字段为域名，第二个字段为公钥算法，第三个字段就是公钥了

# 主动获取某一台服务器的公钥的 fingerprint
$ ssh-keyscan -t {rsa | dsa | ecdsa | ed25519} github.com \
      | ssh-keygen -E sha256 -lf /dev/stdin

# 如果在 ssh-keyscan 的时候指定了 -H 参数，则服务器名将会显示为 Hash 过的字符串
# 比如 ssh-keyscan -H github.com 域名可能会显示为:
#  |1|kHPyrdupxhI41P3kHwrWuAH9Vv4=|kr07SMrFYHr4ZXPeUreL0Htehb4=
# 这个字符串使用 '|' 分割，中间部分为 sha1 算法的 key 的 base64 编码
# 后面部分为 sha1 结果的 base64 编码, 可以通过如下脚本来验证
$ php -r 'echo base64_encode(hash_hmac("sha1", "github.com", base64_decode("kHPyrdupxhI41P3kHwrWuAH9Vv4="), true));'
```

### 自动化脚本

```bash
# 很多时候，我们需要自动在远程执行一些命令，比如:
$ ssh example.com ls /tmp
# 或者，自动 copy 一些文件到远程的服务器上
$ scp -r just-a-test.txt example.com:/path/to/

# 这种情况下，如果是新的服务器，每次都需要手工来确认接受新服务器的公钥
# 我以前是通过 expect 来自动确认
# 现在才知道可以直接通过 ssh-keyscan 来自动获取公钥，保存到 known_hosts
$ ssh-keyscan example.com >> ~/.ssh/known_hosts
# 之后，就可以自动执行脚本了

# 可以通过校验 fingerprint 的方式来校验服务器的公钥是否已经更新
# 获取最新的 example.com 服务器的公钥的 fingerprint
$ ssh-keyscan example.com | ssh-keygen -E {sha256 | md5} -lf /dev/stdin
# 获取 known_hosts 中的公钥的 fingerprint
$ ssh-keygen -F example.com -lf ~/.ssh/known_hosts -E {sha256 | md5}

# 如果 fingerprint 不一致，则可以通过先删除，再添加的方式来更新公钥
$ ssh-keygen -R example.com
$ ssh-keyscan example.com >> ~/.ssh/known_hosts
```

### 参考文档

1. https://www.ssh.com/ssh/
1. https://www.ssh.com/ssh/public-key-authentication
1. https://steemit.com/openssl/@oflyhigh/python-ecdsa-openssl
1. http://www.cnblogs.com/ifantastic/p/3984544.html
1. https://tools.ietf.org/html/rfc4253
