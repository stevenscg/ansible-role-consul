# Ansible Role: Consul

Ansible role installs and configures consul, dnsmasq, and supporting tools.


## Group Vars

In `group_vars/main.yml`:

```yml
# ACL is enabled by default - see tasks/main.yml
consul_acl_enabled: true

# ACL flags - must be defined when ACLs are being used:
consul_acl_datacenter: "default"
consul_acl_default_policy: deny

# ACL rules - when defined, activates the consul-acl.yml task
consul_acl_rules:
  - name: "agent-mgmt"
    type: client
    token: "{{ consul_acl_token_agent_mgmt }}"
    rules:
      - key: ""
        policy: write
      - service: ""
        policy: write

# DNS
consul_dns_recursors:
  # - 169.254.169.253
  - 10.101.0.2
consul_dns_config:
  max_stale: 2s
  service_ttl:
    "*": 0s
    rds: 5s
    elasticache: 5s
```

In `group_vars/secrets.yml`:

```yml
# All nodes within a Consul cluster must share the same encryption key.
# Generate key with "consul keygen"
consul_encrypt_key:

# Master token used by consul servers to bootstrap the ACL system.
consul_acl_master_token:

# An arbitrarily named "management" token for agents with full read
# and write access (see consul_acl_rules)
consul_acl_token_agent_mgmt:

```


## Consul Agent: Server Mode

Example playbook targeting "consul-servers" hosts.

#### Vagrant

```
ansible-playbook consul-servers.yml -e consul_bootstrap_expect=1 -e consul_acl_enabled=false
```

```yml
- hosts: consul-servers
  vars_files:
    - group_vars/main.yml
    - group_vars/secrets.yml

  roles:
    - role: consul
      consul_is_server: true
      consul_datacenter: test
      consul_node_name: vagrant
      consul_acl_enabled: false
      consul_bind_address: "{{ ansible_default_ipv4['address'] }}"
      consul_statsd_address: "127.0.0.1:8125"
      consul_install_dnsmasq: true
      tags: consul
```


#### AWS

```yml
# This uses the EC2 instance tag "class=consul" to identify the consul servers
# during bootstrapping.
- hosts: consul-servers
  vars_files:
    - group_vars/main.yml
    - group_vars/secrets.yml

  roles:
    - role: consul
      consul_is_server: true
      consul_datacenter: default
      consul_node_name: "{{ ansible_ec2_instance_id }}"
      consul_bind_address: "{{ ansible_default_ipv4['address'] }}"
      consul_statsd_address: "127.0.0.1:8125"
      consul_install_dnsmasq: true
      consul_use_retry_join_ec2: true
      consul_servers_via_tag_key: class
      consul_servers_via_tag_value: consul
      consul_acl_token: "{{ consul_acl_token_agent_mgmt | default(omit) }}"
      tags: consul
```


## Consul Agent: Client Mode

Example playbook targeting "web-servers" hosts that also run a consul client.

#### Vagrant

```
ansible-playbook web-servers.yml
```

```yml
- hosts: web-servers
  vars_files:
    - group_vars/main.yml
    - group_vars/secrets.yml

  roles:
    - role: consul
      consul_datacenter: test
      consul_node_name: vagrant
      consul_bind_address: "{{ ansible_default_ipv4['address'] }}"
      consul_statsd_address: "127.0.0.1:8125"
      consul_install_dnsmasq: true
      tags: consul
```


#### AWS

```yml
# This uses the EC2 instance tag "class=consul" to identify the consul servers
# during bootstrapping.
- hosts: web-servers
  vars_files:
    - group_vars/main.yml
    - group_vars/secrets.yml

  roles:
    - role: consul
      consul_datacenter: default
      consul_node_name: "{{ ansible_ec2_instance_id }}"
      consul_bind_address: "{{ ansible_default_ipv4['address'] }}"
      consul_statsd_address: "127.0.0.1:8125"
      consul_install_dnsmasq: true
      consul_use_retry_join_ec2: true
      consul_servers_via_tag_key: class
      consul_servers_via_tag_value: consul
      consul_acl_token: "{{ consul_acl_token_agent_mgmt | default(omit) }}"
      tags: consul
```


## Dependencies

###### Required:

None.

###### Recommended:

Nginx - https://github.com/geerlingguy/ansible-role-nginx


## License

MIT
