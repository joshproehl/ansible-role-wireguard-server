# Wireguard server configured by Ansible

Sets up a wireguard server and creates configs for the specified clients,
and gives you an easy way to fetch/display those configs.

This is primarily designed as a road-warrior style VPN server, so although
Wireguard can be peer to peer, this playbook sets things up as a
hub-and-spoke system with the server as the hub.

Sets up an Unbound DNS resolver on the server and points all clients to that
as their DNS.

Currently designed to run on a Debian10 server, other OS' may not behave
properly. Requires root access.



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
      tags: [ show_wg_client_conf ]             # This tag *must* exist here in order to call the tag from the role
      vars:
        - server_public_ipv4: 1.1.1.1           # Optional, defaults to the server auto-discovering it's publicly available IPv4 address. If the server has multiple, or is being relayed through a proxy, set this manually.
        - wg_port: 44444                        # Optional, defaults to 51820. 
        - server_dns_name: "wg.server.local"    # Optional, if the server needs to be reached via a DNS rather than it's public IPv4 address set this and the client configs will be generated using this address.
        - wg_subnet:
            network: 10.3.2.0
            cidr: "/24"
            server_ip: 10.3.2.1
            wg_conf_path: "/my/conf/path"       # Optional, defaults to /etc/wireguard
            block_client_communication: False   # Optional, defaults to true. If false, does not set IPTables rule that blocks all traffic between clients
        - allowed_ips:                          # Optional, a list of IP/cidr strings which will be added to AllowedIPs for all clients.
          - 10.10.10.0/24
        - wg_clients:
          - name: client1
            wg_ip: 10.3.2.2
            allowed_ips:                        # Optional list of IP/cidr strings which will be added to AllowedIPs for this client, in additional the the global AllowedIPs defined for all clients.
              - 10.0.0.1/24
            send_all_traffic: true              # Optional, defaults to false. If set to true will add 0.0.0.0/0 to AllowedIPs for this client 
            remove: true                        # Optional, defaults to false. If set to true will remove this client from the server, after which you can remove it from the playbook.
            persistent_keepalive: 50            # Optional, defaults to not being added to config. Any value other than false/0 is added to the config.
            iptables_forward_ports:             # Optional list of ports which will be forwarded to this client. No effort is made to prevent configuration errors, in the case of multiple clients requesting the same port the last one handled will win.
              - from: 54321                           # Required, the port on the server to forward
                to: 54321                             # Required, the port to forward to on the client
                proto: tcp                            # Required, must be a valid protocol/match value. (tcp/udp)
                comment: Forward Service to client1   # Optional, defaults to "WGFWD-client_name: ", followed by the given comment if provided.
            block_client_communication: False   # Optional, defaults to false. Creates an iptables rule that specifically allows traffic from this client to the rest of the subnet.
            
```

Once the server is created you can fetch configs for the client using:
`ansible-playbook playbook.yml --tags show_wg_client_conf -e wg_client=client1`

Show a scannable QR code of the generated config by adding `-e show_as_qr=true` to the above command.
