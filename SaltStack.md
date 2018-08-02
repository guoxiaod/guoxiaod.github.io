## SaltStack

### SALT.STATES.USER

#### 添加账号

```saltstack
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

```

### SALT.STATES.SSH_KNOWN_HOSTS

#### 确保指定的域名在 给定的 user 的 known_hosts 文件中

```markdown
anders.just-a-example.com:
  ssh_known_hosts:
    - present
    - user: root
    - enc: ecdsa
    - fingerprint: 'xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx'
    - fingerprint_hash_type: sha256

```
