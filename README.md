# Raspian 12 (bookworm) - USB Tether Router
This is how I turned my Raspberry Pi 4B into a router between my Android (connected via USB tether) and an ethernet device.

This will use eth0 (the on-board NIC) for LAN and any other network device for WAN (eth1, usb0, etc).

LAN (eth0) is statically configured with 192.168.50.1/24 as the IP address and has a DHCP scope of 192.168.50.10-192.168.50.100 (12h lease time).  DHCP also hands out 1.1.1.1 and 1.0.0.1 for the DNS servers.

## Make sure we're all up to date
```
sudo apt update
sudo apt upgrade -y
```

## Set static IP for eth0 (LAN)
```
sudo nmcli con add type ethernet ifname eth0 con-name lan ipv4.address 192.168.50.1/24 ipv4.method manual
sudo nmcli con up lan
```

## Install and configure dnsmasq for DHCP on eth0 (LAN)
```
sudo apt install dnsmasq

cat <<'EOF' | sudo tee /etc/dnsmasq.d/lan.conf >/dev/null
interface=eth0
dhcp-range=192.168.50.10,192.168.50.100,12h
domain-needed
bogus-priv
dhcp-option=3,192.168.50.1
dhco-option=6,1.1.1.1,1.0.0.1
log-dhcp
dhcp-leasefile=/var/lib/misc/dnsmasq.leases
EOF

sudo systemctl enable dnsmasq
sudo systemctl start dnsmasq
```

## Enable IPv4 forwarding
```
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-forward.conf >/dev/null
sudo sysctl -p /etc/sysctl.d/99-forward.conf
```

## Install and configure nftables for NAT
```
sudo apt install nftables

cat <<'EOF' | sudo tee /etc/nftables.conf >/dev/null
#!/usr/sbin/nft -f

flush ruleset

table inet nat {
  chain prerouting {
    type nat hook prerouting priority -100;
  }
  chain postrouting {
    type nat hook postrouting priority 100;
    oifname != "eth0" if saddr 192.168.50.0/24 masquerade;
  }
}

table inet filter {
  chain input {
    type filter hook input priority 0;
    policy accept;
  }
  chain forward {
    type filter hook forward priority 0;
    policy drop;
    
    iifname "eth0" oifname != "eth0" accept;
    iifname != "eth0" oifname "eth0" ct state related,established accept;
  }
  chain output {
    type filter hook output priority 0;
    policy accept;
  }
}
EOF

sudo systemctl enable nftables
sudo systemctl start nftables
```

## Done - Time to test!

* Start by connecting your phone to one of the USB ports on the Pi and turn on tethering.
* Next, connect the Pi's ethernet port to your device
* Try accessing the internet

If that works, reboot the Pi and test again to make sure all services come up as expected.