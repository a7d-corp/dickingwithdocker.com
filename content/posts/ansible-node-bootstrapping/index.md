---
title: "Ansible Node Bootstrapping"
date: "2017-08-04"
categories: 
  - "ansible"
  - "automation"
  - "deployment"
tags: 
  - "ansible"
  - "automation"
---

When you receive a new server, there are a variety of pre-requisites required before Ansible can be used to administrate the host.

Below is my own personal playbook which works for both Debian and RedHat (and derivative) systems.

```
---
# ansible-playbook bootstrap-ansible-target.yml -b -e 'user=user'
- hosts: "{{ host }}"
  remote_user: "{{ user | default('root') }}"
  gather_facts: no

  pre_tasks:
  - name: attempt to update apt's cache
    raw: test -e /usr/bin/apt-get && apt-get update
    ignore_errors: yes

  - name: attempt to install Python on Debian-based systems
    raw: test -e /usr/bin/apt-get && apt-get -y install python-simplejson python
    ignore_errors: yes

  - name: attempt to install Python on CentOS-based systems
    raw: test -e /usr/bin/yum && yum -y install python-simplejson python
    ignore_errors: yes

  - setup:

  tasks:
  - name: Create admin user group
    group:
      name: admin
      system: yes
      state: present

  - name: Ensure sudo is installed
    package:
      name: sudo
      state: present

  - name: Create Ansible user
    user:
      name: ansible
      shell: /bin/bash
      comment: "Ansible management user"
      home: /home/ansible
      createhome: yes

  - name: Add Ansible user to admin group
    user:
      name: ansible
      groups: admin
      append: yes

  - name: Add authorized key
    authorized_key:
      user: ansible
      state: present
      key: "{{ lookup('file', '/etc/ansible/.ssh/id_rsa.pub') }}"

  - name: Copy sudoers file
    command: cp -f /etc/sudoers /etc/sudoers.tmp

  - name: Backup sudoers file
    command: cp -f /etc/sudoers /etc/sudoers.bak

  - name: Ensure admin group can sudo
    lineinfile: 
      dest: /etc/sudoers.tmp
      state: present
      regexp: '^%admin'
      line: '%admin ALL=(ALL) NOPASSWD: ALL'
    when: ansible_os_family == 'Debian'

  - name: Ensure admin group can sudo
    lineinfile:
      dest: /etc/sudoers.tmp
      state: present
      regexp: '^%admin'
      insertafter: '^root'
      line: '%admin ALL=(ALL) NOPASSWD: ALL'
    when: ansible_os_family == 'RedHat'

  - name: Replace sudoers file
    shell: visudo -q -c -f /etc/sudoers.tmp && cp -f /etc/sudoers.tmp /etc/sudoers

  - name: Test Ansible user's access
    local_action: shell ssh ansible@{{ host }} "sudo echo success"
    become: False
    register: ansible_success

  - name: Remove Ansible SSH key from bootstrap user's authorized keys
    lineinfile:
      path: "{{ ansible_env.HOME }}/.ssh/authorized_keys"
      state: absent
      regexp: '^ssh-rsa AAAAB3N'
    when: ansible_success.stdout == "success"
```
