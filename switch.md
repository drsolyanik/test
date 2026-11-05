```
?
help
show version
show system-info
show
```

```
ssh -vvv username@address 
```

```
ansible-galaxy collection list | grep ansible.netcommon
```

```
ansible_connection: ansible.netcommon.network_cli

ansible_user: admin
ansible_password: "{{ vault_switch_ssh_password }}"

ansible_network_os: ""

ansible_become: true
ansible_become_method: enable
ansible_become_password: "{{ vault_switch_enable_password }}"
# Не использовать SSH-ключи
ansible_ssh_private_key_file: ""
ansible_ssh_common_args: >-
  -o PubkeyAuthentication=no
  -o PubkeyAuthentication=no
  -o PreferredAuthentications=password,keyboard-interactive
  -o PasswordAuthentication=yes
  -o KbdInteractiveAuthentication=yes
  -o GSSAPIAuthentication=no
  -o IdentitiesOnly=yes
  -o IdentityFile=/dev/null

# Таймауты
ansible_connect_timeout: 30
ansible_command_timeout: 30
```

```
ansible-playbook -i inventory/switches.ini test_switch.yml --ask-pass -vvv
```


```
---
- name: Test TP-Link connection
  hosts: switches
  gather_facts: false

  tasks:
    - name: Get switch information
      ansible.netcommon.cli_command:
        command: show system-info
      register: switch_info

    - name: Display information
      ansible.builtin.debug:
        var: switch_info.stdout
```

```
ansible-playbook -i inventory/switches.ini test_switch.yml -vvv
```

```
---
- name: Test switch SSH
  hosts: switches
  gather_facts: false

  vars:
    ansible_connection: ssh
    ansible_user: admin
    ansible_password: "PASSWORD"

  tasks:
    - name: Execute command
      ansible.builtin.raw: "show system-info"
      register: result

    - name: Show result
      ansible.builtin.debug:
        var: result.stdout
```
