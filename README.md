# Wireguard server configured by Ansible

Sets up a wireguard server and creates configs for the specified clients,
and gives you an easy way to fetch/display those configs.

Currently designed to run on a Debian10 server, other OS' may not behave properly.
Requires root access.

This is primarily designed as a road-warrior style VPN server.


## Usage
Import this role by adding
```
- src: git+git@github.com:joshproehl/ansible-role-wireguard-server.git
  path: roles
```
to your requirements.yml, and running `ansible-galaxy install -r requirements.yml`.

Here's an example of what you can add to your playbook to create the wireguard server
```
- name: set up Wireguard server
  hosts: wg.mydomain.com
  tags: [ wireguard ]
  tasks:
    - name: wireguard server role
      include_role:
        name: ansible-role-wireguard-server
        apply:
          become: yes                           # This apply must exist if your remote user isn't root
      tags: [ show_wg_client_conf ]             # This tag must exist in order to call the tag from the role
      vars:
        - wg_subnet:
            server_ip: 10.119.81.1
            cidr: "/24"
            allow_client_communication: false   # Optional, defaults to true. If false will create routing rules to prevent client communication
            wg_conf_path: "/my/conf/path"       # Optional, defaults to /etc/wireguard
        - wg_clients:
          - name: client1
            wg_ip: 10.119.81.2
            wg_dns_enabled: true                # Optional, defaults to false
            allowed_ips:
              - 10.0.0.1/24
              - 192.168.1.1/32
            send_all_traffic: true              # Optional, defaults to false. Set to true to create a config which routes all trafic on the client through the server
            remove: true                        # Optional, defaults to false. If set to true will remove this client from the server, after which you can remove it from the playbook.
            persistent_keepalive: 50            # Optional, defaults to 25. Set to false to remove persistent_keepalive entirely from the generated config
            
```

Once the server is created you can fetch configs for the client using:
`ansible-playbook playbook.yml --tags show_wg_client_conf -e wg_client=client1`

Show a scannable QR code of the generated config by adding `-e show_as_qr=true` to the above command.
