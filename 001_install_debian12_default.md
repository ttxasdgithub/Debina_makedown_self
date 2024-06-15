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
### 配置DNS解析

```

vi /etc/resolv.conf



nameserver 168.95.1.1

```


### 確認更新到最新版本

```

sudo apt update
sudo apt upgrade

apt install sudo

```

### 啟用 sudo 

將普通用的加入sudo 群組

1. 通過命令將 使用者 linuxf5 添加到sudo 群組

```
usermod -aG sudo linuxf5

#  更新 sudo 组
newgrp sudo

```

2. 通過編輯 /etc/sudoers 文件將 linuxf5 加入 sudoers 文件中


首先 先通過 su - 命令切換到 root 帳號底下


```

su -

```

然後，輸入 root 帳號密碼後, 通過 vi 編輯 /etc/sudoers 文件 

```

vi /etc/sudoers

```

按 j 鍵 將游標標示移動到 root ALL=(ALL:ALL) ALL 行。

這時候 你可以按下鍵盤上的 yyp 就會自動複製，然後修改下面新的一行

按下 dw 鍵刪除當前 root 單字， 按下i 鍵進入編輯模式，輸入 linuxf5 , 之後按esc鍵 退出編輯模式，按下 w! 鍵 進行強制保存(因為是唯獨檔案)

編輯完成後，就可以通過 su命令切換到 linuxf5 下使用 sudo 命令了


###

```
// 切換到root用户
su -
// 先更新軟件包
apt -y update
apt -y upgrade
// 安装net-tools
apt install -y net-tools

```
