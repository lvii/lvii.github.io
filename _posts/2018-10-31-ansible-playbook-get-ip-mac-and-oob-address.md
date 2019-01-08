---
layout: post
title: "使用 ansible 批量获取 IP MAC OOB 地址对应关系"
category: code
tags: [ansible]
---

# WHY

最近又遇到需要批量获取服务器的 IP、MAC 及 OOB 地址的对应关系

想想还是写个 ansible playbook 来干这事吧

# HOW

## local facts

`gather_facts` 有 2 个 local facts 能收集 IP 及 MAC 信息：

- `ansible_eth0`
- `ansible_default_ipv4`

<https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#local-facts-facts-d>

### playbook

`eth0-mac-and-oob.yml` ：

```
{% raw %}
---
- hosts: all
  gather_facts: yes
  remote_user: root
  tasks:

    - name: IPMI channel
      raw: dmidecode -s system-manufacturer|egrep -iwq 'H3C|Huashan' && echo 8 || echo 1
      register: ipmi_channel

    - name: OOB_IP
      raw: ipmitool lan print {{ ipmi_channel.stdout.strip() }}|awk '/^IP Address[[:space:]]+:/{print $NF}'
      register: oob_ip

    - name: ansible_default_ipv4
      debug: msg="{{ ansible_host }} {{ hostvars[inventory_hostname].ansible_default_ipv4.address }} {{ hostvars[inventory_hostname].ansible_default_ipv4.macaddress }}"

    - name: ansible_eth0.ipv4
      debug: msg="{{ ansible_host }} {{ hostvars[inventory_hostname].ansible_eth0.ipv4.address }} {{ hostvars[inventory_hostname].ansible_eth0.macaddress }} {{ oob_ip.stdout.strip() }}"
{% endraw %}
```

### `ansible_eth0`

    # ansible -i 10.160.155.15, all -m setup -a 'filter=ansible_eth0'

    10.160.155.15 | SUCCESS => {
        "ansible_facts": {
            "ansible_eth0": {
                "active": true,
                "device": "eth0",
                ... ...
                "ipv4": {
                    "address": "10.160.155.15",
                    "broadcast": "10.160.155.255",
                    "netmask": "255.255.255.0",
                    "network": "10.160.155.0"
                },
                "macaddress": "72:85:c4:10:ff:9c",
                "module": "ixgbe",
                "mtu": 1500,
                "pciid": "0000:5f:00.0",
                "phc_index": 0,
                "promisc": false,
                "speed": 10000,
                ... ...
                "type": "ether"
            }
        },
        "changed": false
    }

### `ansible_default_ipv4`

    # ansible -i 10.160.155.15, all -m setup -a 'filter=ansible_default_ipv4'

    10.160.155.15 | SUCCESS => {
        "ansible_facts": {
            "ansible_default_ipv4": {
                "address": "10.160.155.15",
                "alias": "eth0",
                "broadcast": "10.160.155.255",
                "gateway": "10.160.155.254",
                "interface": "eth0",
                "macaddress": "72:85:c4:10:ff:9c",
                "mtu": 1500,
                "netmask": "255.255.255.0",
                "network": "10.160.155.0",
                "type": "ether"
            }
        },
        "changed": false
    }

### which

local facts | problem
:--- | :----
`ansible_eth0` | 不是所有的机器 IP 地址配置在 `eth0` 网卡，可能是 `eth1`、`eth2` 。。。
`ansible_default_ipv4` | 不是所有机器都只有一个 IPv4 地址，可能还有 **Secondary IP**

所有才有 “下策”

## raw shell command

### playbook

```
{% raw %}
---
- hosts: all
  gather_facts: no
  remote_user: root
  tasks:

    - name: IPMI channel
      raw: dmidecode -s system-manufacturer|egrep -iwq 'H3C|Huashan' && echo 8 || echo 1
      register: ipmi_channel

    - name: OOB_IP
      raw: ipmitool lan print {{ ipmi_channel.stdout.strip() }}|awk '/^IP Address[[:space:]]+:/{print $NF}'
      register: oob_ip
      # when: manufacturer.stdout.strip() | regex_search('(H3C|Huashan)')

    - name: MAC
      raw: cat "/sys/class/net/$(ip -o a|awk '/\<{{ ansible_host }}\>/{print $2}')/address"
      register: mac

    - name: IP MAC OOB_IP
      debug: msg="{{ ansible_host }} {{ mac.stdout.strip() }} {{ oob_ip.stdout.strip() }}"
{% endraw %}
```

上面的 playbook 的 task 用的是一陀 shell raw 命令，如果使用 shell 脚本来解析配置能更灵活粗暴。
那样的话需要维护 playbook 外加 script
这样偷懒一下，虽然牺牲了 shel 脚本简单的逻辑，但也解决了 `gather_facts` 局限性，而且 **速度快**

### result

```
# ansible-playbook -i ip-list ~/ip-mac-oob.yml

PLAY [all] ****************************************************************

TASK [IPMI channel] *******************************************************
changed: [10.160.155.15]
changed: [10.160.155.16]

TASK [OOB_IP] *************************************************************
changed: [10.160.155.16]
changed: [10.160.155.15]

TASK [MAC] ****************************************************************
changed: [10.160.155.15]
changed: [10.160.155.16]

TASK [IP MAC OOB_IP] ******************************************************
ok: [10.160.155.15] => {
    "msg": "10.160.155.15 72:85:c4:10:ff:9c 10.150.18.125"
}
ok: [10.160.155.16] => {
    "msg": "10.160.155.16 72:85:c4:10:ff:84 10.150.18.126"
}

PLAY RECAP ****************************************************************
10.160.155.15              : ok=4    changed=3    unreachable=0    failed=0
10.160.155.16              : ok=4    changed=3    unreachable=0    failed=0
```

# reference

<https://docs.ansible.com/ansible/2.5/user_guide/playbooks_filters.html#regular-expression-filters>

[Run an Ansible task only when the variable contains a specific string](https://stackoverflow.com/questions/36496911/run-an-ansible-task-only-when-the-variable-contains-a-specific-string/)

[Getting MAC address from Ansible facts in role](https://stackoverflow.com/questions/40224460/getting-mac-address-from-ansible-facts-in-role)

[Ansible で物理 NIC のリストを取得する方法 2015-07-29](https://qiita.com/daniel-star/items/af34e363f7d0fb7b5eb3)

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
