---
title: "Securing a Raspberry Pi"
date: 2020-04-07T11:01:59-04:00
draft: false
tags:
  - system administration
  - raspberry pi
  - ansible
  - automation
---

Recently, I have been working on an automated water system. I am working on a proof of concept that would allow me to expand into more and more IoT projects for my house. In order to accomplish this, I am going to be running some services on a Raspberry Pi. Now, I know some of you are saying....

![Archer saying do you want hackers because that is how you get hackers.](/blog/images/iot-security.jpeg)

I want to add a firewall and change my network to separate these IoT devices from the rest of my home devices. However, that is something that is more of a lift than I have time for at current. However, I could secure the Raspberry Pi as a transitional step. So let's secure things a little bit. Oh! And I love automating things for later expansion so let's do it using Ansible.

```yaml
# main.yml
---
- hosts: pis
  vars_prompt:
    - name: password
      prompt: "Password?"
    - name: root_password
      prompt: "Root Password Change?"
  vars:
    hostname_prefix: node
    locale: en_US.UTF-8
    timezone: "America/New York"
    ssh: true
    keyboard_layout: 
    username: justin
    ufw_enabled: true
    packages:
      - ufw
      - unattended-upgrades
      - neovim
      - git
      - fail2ban
  become: yes
  tasks:
    - name: update raspi-config
      become: true
      apt:
        name: raspi-config
        update_cache: yes
        state: present
        cache_valid_time: 3600
    - name: get hostname
      shell: "raspi-config nonint get_hostname"
      register: pi_hostname
    - name: change hostname
      shell: "raspi-config nonint do_hostname {{ hostname_prefix }}-{{ groups['pis'].index(inventory_hostname) }}"
      when: pi_hostname.stdout != "{{ hostname_prefix }}-{{ groups['pis'].index(inventory_hostname) }}"
    - name: change locale
      shell: "raspi-config nonint do_change_locale {{ locale }}"
    - name: change timezone
      shell: "raspi-config nonint do_change_timezone {{ timezone }}"
    - name: change keyboard layout
      debug:
        msg: "TODO: Change the keyboard layout"
    - name: check ssh
      shell: "raspi-config nonint get_ssh"
      register: ssh_status
      changed_when: False
    - name: enable ssh
      shell: "raspi-config nonint do_ssh 0"
      when: ssh and (ssh_status.stdout != '1')
    - name: disable ssh
      shell: "raspi-config nonint do_ssh 1"
      when: (not ssh) and (ssh_status.stdout != '0')
    - name: add the user '{{ name }}' with a bash shell
      user:
        name: "{{ username }}"
        shell: /bin/bash
        groups: 
          - adm
          - sudo
        append: yes
        password: "{{ password  | password_hash('sha512') }}"
    - name: lock the 'pi' user
      user:
        name: pi
        expires: -1
    - name: change root password
      user:
        name: root
        password: "{{ root_password | password_hash('sha512') }}"
    - name: install packages
      apt:
        name: "{{ item }}"
        state: latest
        update_cache: yes
      loop: "{{ packages }}"
    - name: download the docker script
      get_url:
        url: "https://get.docker.com"
        dest: /tmp/docker.sh
    - name: install docker
      shell: "bash /tmp/docker.sh"
      become: yes
      notify:
        - cleanup docker
    - name: add {{ username }} to the docker group
      user:
        name: "{{ username }}"
        group: docker
        append: yes
    - name: configure unattended upgrades
      lineinfile:
        path: /etc/apt/apt.conf.d/50unattended-upgrades
        line: '"origin={{ item }},codename=${distro_codename},label={{ item }}"'
        loop:
          - Raspbian
          - Raspberry Pi Foundation
    - name: configure ufw
      ufw:
        comment: SSH
        rule: allow
        port: 22
        proto: tcp
      when: ufw_enabled
    - name: restrict ssh
      lineinfile:
        path: /etc/ssh/sshd_config
        line: "AllowUsers {{ username }}"
    - name: protect against brute force
      blockinfile:
        path: /etc/fail2ban/jail.local
        block: |
          [DEFAULT]
          bantime = 1h
          banaction = ufw

          [sshd]
          enabled = true
  handlers:
    - name: cleanup docker
      file:
        name: /tmp/docker.sh
        state: absent
```

Now that is a long single playbook and probably doesn't address every security concern. Moreover, I still need to work on the keyboard layout change so I know what key I am pressing. I hope to test this a little more and add support for `dietpi` as well so that I can have a minimal image running on these RPi devices. 

Take a look at the project: [Secure a Raspberry Pi](https://github.com/justin1dennison/securing-a-raspberry-pi)

Thanks for reading. I hope to make changes and improvements as I receive feedback and learn more about my setup. 

