# NA HW1

## Table of Content
* [Basic Setting](#basic-setting)
* [WireGuard VPN](#wireguard-vpn)
* [SSH](#ssh)
* [Internal Network Interface](#internal-network-interface)
* [DHCP](#dhcp)
* [NAT](#nat)
* [Firewall](#firewall)
* [Debug Tools](#debug-tools)

## Basic Setting

1. 設定 sudo user
    
    ```bash
    su - # 切換到 root
    apt update && apt install sudo -y # 確認 sudo 是否已安裝
    usermod -aG sudo [USERNAME]
    su - [USERNAME] # 或是重新登入
    ```
    
2. 安裝基本套件
    
    ```bash
    sudo apt install net-tools iptables vim -y
    ```
    
3. Mount Guest Additions ISO (Optional)
    1. 安裝 Guest Additions  需要的 package
        
        ```bash
        sudo apt install -y build-essential dkms linux-headers-$(uname -r)
        ```
        
    2. 在 VirtualBox 視窗中，點擊 "Devices" → "Insert Guest Additions CD Image..."。
    3. Mount the CD
        
        ```bash
        sudo mount /dev/cdrom /media/cdrom
        ```
        
    4. 切換到 Guest Additions CD 目錄，執行安裝腳本：(以 Debian 為例)
        
        ```bash
        cd /media/cdrom0
        sudo ./VBoxLinuxAdditions.run
        ```
        

---

## WireGuard VPN

1. 安裝 WireGuard
    
    ```bash
    sudo apt update
    sudo apt install -y wireguard
    ```
    
2. 設定 `wg0` interface
    
    ```bash
    sudo vim /etc/wireguard/wg0.conf
    ```
    
3. 貼上 Judge 給的 `wg0.conf` 檔，接著啟動 WireGuard
    
    ```bash
    sudo wg-quick up wg0
    ```
    
4. 開機自動啟動
    
    ```bash
    sudo systemctl enable wg-quick@wg0
    ```
    

---

## SSH

1. 安裝 SSH Server
    
    ```bash
    sudo apt install -y openssh-server
    sudo systemctl enable ssh
    ```
    
2. 確認安裝成功：
    
    ```bash
    systemctl status ssh
    ```
    

---

## Internal Network Interface

1. 設定 `enp0s8` 為 `192.168.17.254`
    
    ```bash
    sudo ip addr add 192.168.17.254/24 dev enp0s8
    sudo ip link set enp0s8 up
    ```
    
2. 啟用 `systemd-networkd`
    
    ```bash
    sudo systemctl enable --now systemd-networkd
    ```
    
3. 建立 `systemd-networkd` 設定檔
    
    ```bash
    sudo vim /etc/systemd/network/10-enp0s8.network
    ```
    
4. 加入：
    
    ```
    [Match]
    Name=enp0s8
    
    [Network]
    Address=192.168.17.254/24
    ```
    
5. 重啟網路
    
    ```bash
    sudo systemctl restart systemd-networkd
    ```
    

---

## DHCP

1. 安裝 DHCP Server
    
    ```bash
    sudo apt install -y isc-dhcp-server
    ```
    
2. 在 `/etc/dhcp/dhcpd.conf` 中加上:
    
    ```
    subnet 192.168.17.0 netmask 255.255.255.0 {
        range 192.168.17.111 192.168.17.222;
        option routers 192.168.17.254;
        option domain-name-servers 8.8.8.8, 8.8.4.4;
        default-lease-time 600;
        max-lease-time 7200;
    
        host agent {
            hardware ethernet 08:00:27:44:92:EA; # Agent VM Mac
            fixed-address 192.168.17.234;
        }
    }
    ```
    
3. 編輯 `/etc/default/isc-dhcp-server`：
    
    ```
    INTERFACESv4="" -> INTERFACESv4="enp0s8" 
    ```
    
4. 重新啟動 DHCP Server
    
    ```bash
    sudo systemctl restart isc-dhcp-server
    ```
    
5. 設定開機自動啟動
    
    ```bash
    sudo systemctl enable isc-dhcp-server
    ```
    

---

## NAT

```bash
# Traffic from Private zone to Internet should be NAT masqueraded.
sudo iptables -t nat -A POSTROUTING -s 192.168.17.0/24 -o enp0s3 -j MASQUERADE

# Traffic from Private zone to VPN zone should not be NAT masqueraded.
sudo iptables -t nat -A POSTROUTING -s 192.168.17.0/24 -d 10.113.0.0/16 -o wg0 -j ACCEPT
```

讓 NAT 設定在開機時自動載入：

```bash
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

---

## Firewall

```bash
# By default, all connections from Internet and VPN zone to Private zone should be rejected.
sudo iptables -A FORWARD -i enp0s3 -d 192.168.17.0/24 -j DROP
sudo iptables -A FORWARD -i wg0 -d 192.168.17.0/24 -j DROP

# By default, all connections from Private zone to VPN zone should be rejected.
sudo iptables -A FORWARD -i enp0s8 -o wg0 -j DROP

# SSH connections from anywhere to "Agent" are allowed.
sudo iptables -I FORWARD -p tcp --dport 22 -d 192.168.17.234 -j ACCEPT
sudo iptables -I FORWARD -i enp0s8 -o wg0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# SSH connections from VPN zone to "Router" are not allowed.
sudo iptables -A INPUT -i wg0 -p tcp --dport 22 -j DROP

# ICMP connections from anywhere to anywhere are allowed.
sudo iptables -I INPUT -p icmp -j ACCEPT
sudo iptables -I FORWARD -p icmp -j ACCEPT

# Allow HTTP (80)/HTTPS (443) from Private zone to VPN zone.
sudo iptables -A FORWARD -i enp0s8 -o wg0 -p tcp --dport 80 -j ACCEPT
sudo iptables -A FORWARD -i enp0s8 -o wg0 -p tcp --dport 443 -j ACCEPT
```

記得存檔

```bash
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

---

## Debug Tools

```bash
sudo iptables -L -v -n # 確認防火牆相關規則
sudo iptables -t nat -L -v -n # 確認 NAT 相關規則
ip a # 查看 interface 狀態
```