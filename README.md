# network-automation

# Mikrotik Overview

This will create the following on a MikroTik router:

**NETWORKS**
| VLAN  | ID  | IPv4 Range     | IPv6  | Notes
|  ---  | --- |      ---       |  ---  | ---
| USER  | 5   | 192.168.5.0/24 | Yes   | Trusted network for user devices            |
| GUEST | 6   | 192.168.6.0/24 | Yes   | Access to internet and NAS files            |
| IOT   | 7   | 192.168.7.0/24 | No    | No internet access, restricted access       |
| NTWK  | 9   | 192.168.9.0/24 | No    | No internet access, restricted access       |
| INFRA | 99  | 192.168.0.0/24 | Yes   | Internet access granted to specific servers |

* Wireguard Range: 172.16.32.0/24
* IPv6 is assigned from ISP provided prefix

**INTERFACES**
* ether1: WAN port
* ether2: Trunk port to HPe 1910 switch
* ether3: unused
* ether4: unused
* ether5: not part of bridge, backdoor for when i break config. IP is 192.168.88.1
* wireguard: VPN

**INTERFACE LISTS**

These simplify firewall rules.
* EVERYTHING_INTERNAL: ALL internal interfaces or VLANS
* BACKDOOR: physical interfaces allowing backdoor access - just ether5
* ALL_VLANS: Contains all VLAN interfaces
* ALLOW_INTERNET_RANGES: specific interfaces allowed to access internet - USER, GUEST VLANS and WireGuard VPN only
* WAN: WAN interfaces - ether1

**SERVICES**
* NTP: Announcing time to anyone who asks.
* DHCP Server: Available for for VLANS 5,6,7,9,99. IPv4 only
* Syslog: Logging specific topics to local syslog server. Forwarded into splunk.
* DNS: Provides DNS functionality. Also used to resolve various internal services via static entries
* Dynamic DNS: Using built in one provided by Mikrotik

**FIREWALL RULES**

Available for IPv4 and IPv6.

Both leverage interface rules and address lists.
From a conceptual viewpoint we are preventing from the WAN any dodgy traffic, BOGONS & inbound traffic not part of already established connection. Between networks we are preventing multicast or broadcast and denying/allowing specific use cases.

Lastly we are leveraging prefiltering to apply before routing (forwarding) to minimise DDOS type attacks from the internet and also using fasttrack so established/related packets don't need to be tracked by the router. Fasttrack typically reduces CPU usage - important as the router doesnt have the fastest CPU.

**FURTHER SECURITY MEASURES**

Aside from firewall rules,
* Unused services such as api (non-https), telnet, http, ftp are disabled.
* Enabled SSH strong crypto
* Disabled CDP, LLDP, MNDP (neighbour discovery)

## Starting from Scratch

**These steps assume you have installed the winbox application**

To start from scratch with no default configuration, select **System** -> **Reset Configuration** -> tick **No Default Configuration** -> select **Reset Configuration**

After the device has restarted follow steps:
1. Connect laptop to ether5. Configure IP address of laptop as 192.168.88.2/24 (no GW)
2. Launch winbox and connect to mac-address
3. Login to device and set admin password
4. Next run the following commands to create ansible user/group and configure ether5
```
/user/group/add name=ansible policy=ssh,read,write,policy,test,api
/user/add name=ansible password="<<password_here>" comment="ansible automation user" group=ansible

/ip address/add interface=ether5 address=192.168.88.1/24 network=192.168.88.0
```

5. Update **/inventory/inventory.yml**, set **ansible_host** to 192.168.88.1 and update passwords
6. Launch ansible-playbook with `make mikrotik-deploy`
    1. **What it does**: Ansible on it's first launch runs the api-setup.yml playbook which connects via ssh to device and generates a certificate. Then this is used to configure https. Once this is done further configuration takes place using the API.

7. After running playbook for the first time update **inventory.yml** with correct device IP. Future runs don't require being connected to ether5

8. As a final step, confirm NTP time synchronization is occuring

## Outstanding

* API is unable to enable graphing
* Investigate initial time synchronization further
* Issues with redeploying to an existing device which has been reset - need to clear the ssh known hosts file.
* Manage prometheus user in code.

## References

* FW Rules: https://help.mikrotik.com/docs/display/ROS/Building+Advanced+Firewall
* Router Security: https://help.mikrotik.com/docs/display/ROS/Securing+your+router
* Ansible Doco: https://docs.ansible.com/ansible/devel/collections/community/routeros/

-----
-----
# Unifi Overview

This will create the following SSID's on a unifi network appliance:

| SSID           | Network Range | VLAN | Notes
|  ---           | ---           |  --- |  ---
| THE-VOID       | USERS         | 5    |
| THE-VOID-GUEST | GUEST         | 6    |
| THE-VOID-IOT   | IOT           | 7    | client isolation enabled

As well this configures all settings on the unifi appliance.

## Starting from Scratch

1. Create an ansible network account.
    1. Change interface to legacy mode via **Settings** -> **System** -> **Advanced** -> Set **Interface** to *Legacy*
    2. Create ansible account. Go to **Settings** -> **Admins** -> **ADD NEW ADMIN**. Use Settings:
        1. **Invite to UniFi Network:** Manually set and share the password
        2. **Untick** "Require the user to change the password"
        3. **Role:** Super Administrator

2. Update user, password and wireless network passwords in *inventory.yml*
3. Deploy by running `make unifi-deploy`

## Notes

Redeploying wifi settings will cause wifi & AP to go offline. Once wifi online re-run set-inform command.

Login to AP via `ssh -o RSAMinSize=1024 admin@AP`

## References

* API Overview: https://ubntwiki.com/products/software/unifi-controller/api
* API Reverse Engineered: https://github.com/Art-of-WiFi/UniFi-API-client/blob/master/src/Client.php
* Forum post stating ansible possibility: https://community.ui.com/questions/Its-possible-to-automate-Unifi-Controller-configs/60c753ee-ec19-43f3-b758-cfce6eae6162

-----
-----
# Cisco Switch Overview

This will create a config for a Cisco switch.

## Starting from Scratch
Connect via console cable to apply initial settings.

1. Enter config mode
    1. `enable`
    2. `config t`
2. Create an ansible network account (type 8): `username ansible privilege 15 algorithm-type sha256 <<password>>`
3. Create host settings `ip domain-name <<domain-name>>`
4. Generate RSA key
    1. `crypto key generate rsa`
    2. Specify `2048`
5. Enable SSH:
    ```
    line vty 0 4
    login local
    transport input ssh
    ```
6. Enable management interface
    ```
    interface vlan9
    ip address 192.168.9.10 255.255.255.0
    no shutdown
    ```
7. Save config `copy running-config startup-config`
8. Update `ansible_ssh_pass` in *inventory.yml*
9. Deploy by running `make switch-deploy`
