# Knowledge article for PhotonOS

## Change password to never expire

`chage –I -1 –m 0 –M  99999 –E -1 root`

## Configure initial static IP

`vi /etc/systemd/network/10-static-eth0.network`

    [Match]
    Name=eth0

    [Network]
    Address=<IP-Address>/<IP-Network>
    Gateway=<IP-Gateway>
    DNS=<IP-DNSServer>

`chmod 655 /etc/systemd/network/*`

## Enable root login via SSH

Change content in `/etc/ssh/sshd_config` from

    PermitRootLogin no

to

    PermitRootLogin yes

and restart sshd with `systemctl restart sshd`

## Persistently set firewall entries with iptables

Set desired entries directly in iptables, e.g. for a 2 network router:

    iptables -P INPUT ACCEPT
    iptables -P FORWARD ACCEPT
    iptables -P OUTPUT ACCEPT
    iptables -t nat -F
    iptables -t mangle -F
    iptables -F
    iptables -X
    iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT
    iptables -A FORWARD -i eth0 -o eth2 -m state --state RELATED,ESTABLISHED -j ACCEPT
    iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
    iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT

reload the current configuration with `systemctl restart iptables`

and save the configuration for after a restart with `iptables-save > /etc/systemd/scripts/ip4save`
