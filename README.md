# clu-ansible-roles

Ansible roles used in cludus

# Setup

This setup shows how to install the collection in a project to a directory .collections but the dir name can be anything you want.

In ansible.cfg add the collections_paths variable

    [defaults]
    inventory = inventory.yml
    interpreter_python = auto_silent
    host_key_checking = False
    ...
    collections_paths = ./.collections/ansible_collections

Run the following command with ansible-galaxy to install the collection to the local .collections folder

    ansible-galaxy collection install git+https://github.com/cludus/clu-ansible-commons.git -p ./.collections

Using requirements.yml file is also posible

    collections:
    - name: https://github.com/cludus/clu-ansible-commons.git
      type: git

then install it by using 

    ansible-galaxy collection install -r requirements.yml -p ./collections

# Usage

Here is a simple play book using some roles
    
    - name: update routers
      gather_facts: true
      hosts: routers
      become: true
      old-roles:
        - role: cludus.commons.dns
        - role: cludus.commons.router
    
    - name: update servers
      gather_facts: true
      hosts: pcs
      become: true
      old-roles:
        - role: cludus.commons.dns
        - role: cludus.commons.routes
    
    - name: update vpns
      gather_facts: true
      hosts: vpns
      become: true
      roles:
        - role: cludus.commons.dns
        - role: cludus.commons.routes
        - role: cludus.commons.vpn
        - role: cludus.commons.router

All the parameters for the roles can be provided in the inventory, we recommend using yml to create the inventory and put all configs there

    pcs:
      hosts:
        pc-a1:
          ansible_host: 192.168.22.100
          ansible_user: vagrant
          routes:
            - net: 10.22.0.0
              mask: 255.255.0.0
              next_hop: 10.22.2.2
        pc-b1:
          ansible_host: 192.168.22.101
          ansible_user: vagrant
          routes:
            - net: 10.22.0.0
              mask: 255.255.0.0
              next_hop: 10.22.3.2
    routers:
      hosts:
        r-a1:
          ansible_host: 192.168.22.104
          ansible_user: vagrant
          networks:
            - ip: 10.22.2.1
              suffix: 24
            - ip: 10.22.10.5
              suffix: 30
            - ip: 10.22.10.17
              suffix: 30
        r-a2:
          ansible_host: 192.168.22.105
          ansible_user: vagrant
          networks:
            - ip: 10.22.10.6
              suffix: 30
            - ip: 10.22.10.21
              suffix: 30
            - ip: 10.22.10.25
              suffix: 30
    
    vpns:
      hosts:
        vpn1:
          ansible_host: 192.168.22.112
          ansible_user: vagrant
          routes:
            - net: 10.22.0.0
              mask: 255.255.0.0
              next_hop: 10.22.2.2
          vpn:
            name: vpn1
            address: 10.9.0.2/32
            endpoint: 10.22.2.11
            listening_port: 36582
            private_key: +Mh/4W2NkJL1ZR3XgmbTLiBOZojsk3qwDT1H+UL9OFk=
            public_key: Hz4bSljUJu7dKNojY9gAx6CtOJynSS7FCQ2WPZVfuEI=
            peers:
              - name: vpn2
              - name: vpn3
        vpn2:
          ansible_host: 192.168.22.113
          ansible_user: vagrant
          routes:
            - net: 10.22.0.0
              mask: 255.255.0.0
              next_hop: 10.22.4.2
          vpn:
            name: vpn2
            address: 10.9.0.3/32
            listening_port: 36582
            endpoint: 10.22.4.11
            private_key: +K5nfFf8Wc37WSC8AiF2kKc+IMCbZMzcXF5F5N4voH8=
            public_key: Jhqylbk7pQSB/t8CxkqxYWBKs58KO/hn8Sh8f0PlR2s=
            peers:
              - name: vpn1
              - name: vpn4****

# Rotating wiregard keys

To rotate wireguard keys use the following script

    #!/bin/bash
    
    function genkeys {
      prvkey=$(wg genkey)
      pubkey=$(echo "$prvkey" | wg pubkey)
    
      yq -i e ".vpns.hosts.$1.vpn.public_key = \"$pubkey\"" inventory.yml
      yq -i e ".vpns.hosts.$1.vpn.private_key = \"$prvkey\"" inventory.yml
    }
    
    yq '.vpns.hosts | keys[]' inventory.yml | while IFS= read -r host; do
      genkeys $host
    done

