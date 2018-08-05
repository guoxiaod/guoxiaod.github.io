## SaltStack

### SALT.STATES.USER

#### 添加账号


```salt
# user.sls -- begin {% raw %}
{%
set users = [
  {
    'login': 'anders', 'uid': 1000, 'gid': 100, 'sudo': True,
    'password': '$6$xxxxxxxx$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
  },
  {
    'login': 'anders2', 'uid': 1000, 'gid': 100, 'sudo': True,
    'password': '$6$xxxxxxxx$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
  }
]
%}

{% for user in users %}
{{ user.login }}:
  user.present:
    - shell: /bin/bash
    - password: {{ user.password }}
    - gid: {{ user.gid }}
    - uid: {{ user.uid }}
    - createhome: True
{% if user.sudo %}
    - groups:
      - wheel
{% endif %}
{% endfor %}
# user.sls -- end {% endraw %}
```

上面提到的 password 并不是密码明文，而是密码明文使用特定算法处理的哈希串，可以使用如下的脚本来生成:

```python
## Python 代码
## 使用交互式的方式生成 password 
$ python -c 'import crypt,getpass;pw=getpass.getpass();print(crypt.crypt(pw) if (pw==getpass.getpass("Confirm: ")) else exit())'
## 直接在脚本中指定明文密码, 生成 password
$ python -c 'import crypt,getpass;pw="12345678";print(crypt.crypt(pw))'
```

```php
## PHP 代码
$ php -r 'echo crypt("yourpassword", "$6$" . base64_encode(random_bytes(6)) . "$");'
```

### SALT.STATES.SSH_KNOWN_HOSTS

#### 确保指定的域名在 给定的 user 的 known_hosts 文件中

```salt
anders.just-a-example.com:
  ssh_known_hosts:
    - present
    - user: root
    - enc: ecdsa
    - fingerprint: 'xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx'
    - fingerprint_hash_type: sha256

```

### 参考文档

1. https://docs.saltstack.com/en/latest/ref/states/all/salt.states.ssh_known_hosts.html
1. https://docs.saltstack.com/en/latest/ref/states/all/salt.states.user.html
1. http://www.cnblogs.com/f-ck-need-u/p/6089869.html
