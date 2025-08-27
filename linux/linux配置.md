##### netplan 静态 IP 设置

```yaml
# /etc/netplan/*.yaml
network:
    ethernets:
        default:
            dhcp-identifier: mac
            dhcp4: true
            match:
                macaddress: 52:54:00:e8:ec:24
        extra0:
            dhcp4: no
            addresses: [192.168.1.50/24]
            routes:
              - to: default
                via: 192.168.1.1
            nameservers:
                addresses: [114.114.114.114, 8.8.8.8]
            match:
                macaddress: 52:54:00:09:2a:fa
    version: 2
```

