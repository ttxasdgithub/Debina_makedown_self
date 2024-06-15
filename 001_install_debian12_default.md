# 安裝 debain 12 
### 001_install_debian12_default.md

```
vi /etc/network/interfaces

# The primary network interface
allow-hotplug ens192
iface ens192 inet static
address 192.168.0.37
netmask 255.255.255.0
gateway 192.168.0.2

```
