# NA HW2

## Table of Content
* [Router](#router)
* [Authoritative DNS Server](#authoritative-dns-server)
    * [1. 安裝 bind9](#1-安裝-bind9)
    * [2. 建立 zone file](#2-建立區域檔-zone-file)
    * [3. 啟用 DNSSEC &amp; 產生金鑰 (KSK + ZSK)](#3-啟用-dnssec--產生金鑰-ksk--zsk)
    * [4. 離線簽署](#4-離線簽署)
* [Resolver](#resolver)
* [Debug Tools](#debug)
* [Reference](#reference)

## Router

- 設定 DHCP，assign IP 給 Authoritative DNS Server 和 Resolver，將 `/etc/dhcp/dhcpd.conf` 改成:
    
    ```
    subnet 192.168.17.0 netmask 255.255.255.0 {
        range 192.168.17.111 192.168.17.222;
        option routers 192.168.17.254;
        option domain-name-servers 8.8.8.8, 8.8.4.4;
        default-lease-time 600;
        max-lease-time 7200;
    
        host agent {
            hardware ethernet 08:00:27:44:92:EA; # Agent VM MAC
            fixed-address 192.168.17.234;
        }
    
    		# NEW: add DNS servers IP assignment
        host authoritative-dns {
            hardware ethernet 08:00:27:D0:53:5E; # Authoritative DNS Server VM MAC
            fixed-address 192.168.17.53; 
        }
    
        host resolver {
            hardware ethernet 08:00:27:38:6D:B6; # Resolver VM MAC 
            fixed-address 192.168.17.153;
        }
    }
    ```
    
- 重啟 DHCP server
    
    ```bash
    sudo systemctl restart isc-dhcp-server
    ```
    
- 允許 DNS, HTTPS, TLS 封包
    
    ```bash
    sudo iptables -I FORWARD  -p tcp --sport 53 -j ACCEPT
    sudo iptables -I FORWARD  -p tcp --dport 53 -j ACCEPT
    sudo iptables -I FORWARD  -p udp --sport 53 -j ACCEPT
    sudo iptables -I FORWARD  -p udp --dport 53 -j ACCEPT
    sudo iptables -I FORWARD  -p tcp --sport 443 -j ACCEPT
    sudo iptables -I FORWARD  -p tcp --dport 443 -j ACCEPT
    sudo iptables -I FORWARD  -p tcp --sport 853 -j ACCEPT
    sudo iptables -I FORWARD  -p tcp --dport 853 -j ACCEPT
    ```
    

## Authoritative DNS Server

> [!Note]  
> 一開始安裝 package 時先不要用 internal network，會被 router 防火牆檔掉

### **1. 安裝 `bind9`**

- 安裝 `bind9`
    
    ```bash
    sudo apt install bind9 -y
    ```
    
- 確保開機後會自動開啟
    
    ```bash
    sudo systemctl enable named
    ```
    
    > `bind9` 是 alias，`named` 才是服務名稱，可以用以下指令確認
    > 
    
    ```bash
    systemctl list-unit-files | grep named
    systemctl list-unit-files | grep bind9
    ```
    

### **2. 建立區域檔 (zone file)**

1. 建立 `17.nasa` zone file，新增 `/etc/bind/db.17.nasa` 檔案，並填入：
    
    ```
    $TTL 86400
    $ORIGIN 17.nasa.
    @   IN  SOA ns1.17.nasa. root.17.nasa. (
                2025031801  ; Serial (日期 + 版本)
                7200        ; Refresh
                3600        ; Retry
                1209600     ; Expire
                86400 )     ; Minimum TTL
    
    ; 掛載 Nameserver
    @       IN  NS  ns1.17.nasa.
    ns1     IN  A   192.168.17.53
    dns     IN  A   192.168.17.153
    whoami  IN  A   10.113.17.1
    ```
    
2. 建立**反向解析 (Reverse Lookup)**，新增 `/etc/bind/db.192.168.17` 檔案，並填入：
    
    ```
    $TTL 86400
    $ORIGIN 17.168.192.in-addr.arpa.
    @   IN  SOA ns1.17.nasa. root.17.nasa. (
                2025031801
                7200
                3600
                1209600
                86400 )
    
    ; 掛載 Nameserver
    @       IN  NS  ns1.17.nasa.
    53      IN  PTR ns1.17.nasa.
    153     IN  PTR dns.17.nasa.
    ```
    
3. 使用 `named-checkzone` 驗證
    
    ```bash
    named-checkzone 17.nasa. /etc/bind/db.17.nasa
    ```
    
    如果回傳 `OK`，表示語法正確。
    
4. 在 `/etc/bind/named.conf.local` 定義區域 (尚未簽名時)，加入底下段落
    
    ```
    zone "17.nasa" {
        type primary;
        file "/etc/bind/db.17.nasa";
    };
    
    zone "17.168.192.in-addr.arpa" {
        type primary;
        file "/etc/bind/db.192.168.17";
    };
    ```
    
5. 重開 `bind9`
    
    ```bash
    sudo systemctl restart bind9
    ```
    

### 3. 啟用 DNSSEC & 產生金鑰 (KSK + ZSK)

1. 開啟 `/etc/bind/named.conf.options` 改成以下內容
    
    ```
    options {
    		directory "/var/cache/bind";
    		dnssec-validation yes;
    		allow-query { any; };
    		listen-on { any; };
    		listen-on-v6 { any; };
    		recursion no;
    		forwarders { };
    };
    ```
    
2. 生成 DNSSEC keys pair
    - 生成 ZSK (Zone Signing Key)
        
        ```bash
        cd /etc/bind
        sudo dnssec-keygen -a ECDSAP256SHA256 -b 256 -n ZONE 17.nasa
        sudo dnssec-keygen -a ECDSAP256SHA256 -b 256 -n ZONE 17.168.192.in-addr.arpa
        ```
        
        output:
        
        ```bash
        Generating key pair.
        K17.nasa.+013+49076
        ```
        
        ```
        Generating key pair.
        K17.168.192.in-addr.arpa.+013+63351
        ```
        
    - 生成 KSK (Key Signing Key)
        
        ```bash
        sudo dnssec-keygen -a ECDSAP256SHA256 -b 256 -n ZONE -f KSK 17.nasa
        sudo dnssec-keygen -a ECDSAP256SHA256 -b 256 -n ZONE -f KSK 17.168.192.in-addr.arpa
        ```
        
        output:
        
        ```bash
        Generating key pair.
        K17.nasa.+013+41875
        ```
        
        ```
        Generating key pair.
        K17.168.192.in-addr.arpa.+013+15389
        ```
        
3. 確保4把 key 中都有 TTL 的部分
    - `17.nasa.`
        
        ```
        17.nasa. **3600** IN DNSKEY 256 3 13 Kr/DnFm5pHfTvR0X80Hour8bZrwpyX8xb1rt94P+iDC79yvGlHJBcV2S 1ZalS9jUoKVQiG2WFRuRwQ1WDRE7AQ==
        ```
        
        ```
        17.nasa. **3600** IN DNSKEY 257 3 13 34w/mpYjrPo2aSFlwUsPbLqvL8rTIBWYrteR3Ho/EeeWF/mp3fPPvaUA Eq9oam2YcVdfYdpxxrfwHMA4iWz94Q==
        ```
        
    - 17.168.192.in-addr.arpa
        
        ```
        17.168.192.in-addr.arpa. **3600** IN DNSKEY 257 3 13 WkzQfpeFXH/zxrRHsdByYLfg10KPcIprJu0xeqecwxCtALL2Qk0Oz3t2 0b6CQ9byYSxcPfu5dwvzlwOHzReVBQ==
        ```
        
        ```
        17.168.192.in-addr.arpa. **3600** IN DNSKEY 256 3 13 1fF3w6SQpdimJ50jpB4RGMuy4ywDnCY4xjM1E4iCOvEnB02/W+LMvYDV HSL/rqh/o2vxgjqVzHwZYdXZ999ksw==
        ```
        
4. 確保鑰匙權限
    
    ```bash
    sudo chown bind:bind K17.nasa.* 
    sudo chmod 600 K17.nasa.*
    
    sudo chown bind:bind K17.168.192.in-addr.arpa.*
    sudo chmod 600 K17.168.192.in-addr.arpa.*
    ```
    

### 4. 離線簽署

1. 在 `/etc/bind/db.17.nasa` 加入 DNSSEC record，在檔案底部加入 DNSKEY record
    
    ```bash
    $INCLUDE /etc/bind/K17.nasa.+013+49076.key
    $INCLUDE /etc/bind/K17.nasa.+013+41875.key
    ```
    
    確保 `dnssec-signzone` 一定能讀到 DNSKEY，不知道為什麼輸入指令的時候指定都讀不到
    
    一樣在 `/etc/bind/db.192.168.17`  在檔案底部加入 DNSKEY record：
    
    ```bash
    $INCLUDE /etc/bind/K17.168.192.in-addr.arpa.+013+63351.key
    $INCLUDE /etc/bind/K17.168.192.in-addr.arpa.+013+15389.key
    ```
    
2. 使用
    
    ```bash
    sudo dnssec-signzone -A -3 aabbccdd -N increment -o 17.nasa. -t db.17.nasa 
    ```
    
    - `3 aabbccdd` → NSEC3 + salt
    - `o 17.nasa.` → zone 名稱 (帶尾巴 `.`)
    - 會產生 `db.17.nasa.signed`
    
    Output:
    
    ```
    Verifying the zone using the following algorithms:
    - ECDSAP256SHA256
    Zone fully signed:
    Algorithm: ECDSAP256SHA256: KSKs: 1 active, 0 stand-by, 0 revoked
                                ZSKs: 1 active, 0 stand-by, 0 revoked
    db.17.nasa.signed
    Signatures generated:                       12
    Signatures retained:                         0
    Signatures dropped:                          0
    Signatures successfully verified:            0
    Signatures unsuccessfully verified:          0
    Signing time in seconds:                 0.036
    Signatures per second:                 333.333
    Runtime in seconds:                      0.076
    ```
    
    ```bash
    sudo dnssec-signzone -A -3 aabbccdd -N increment -o 17.168.192.in-addr.arpa. -t db.192.168.17 
    ```
    
    Output:
    
    ```
    168.192.in-addr.arpa. -t db.192.168.17 
    Verifying the zone using the following algorithms:
    - ECDSAP256SHA256
    Zone fully signed:
    Algorithm: ECDSAP256SHA256: KSKs: 1 active, 0 stand-by, 0 revoked
                                ZSKs: 1 active, 0 stand-by, 0 revoked
    db.192.168.17.signed
    Signatures generated:                       12
    Signatures retained:                         0
    Signatures dropped:                          0
    Signatures successfully verified:            0
    Signatures unsuccessfully verified:          0
    Signing time in seconds:                 0.020
    Signatures per second:                 600.000
    Runtime in seconds:                      0.036
    ```
    
3. 修改 BIND 配置使用已簽名檔，在 `named.conf.local`:
    
    ```
    zone "17.nasa" {
        type primary;
        file "/etc/bind/db.17.nasa.signed";
    };
    
    zone "17.168.192.in-addr.arpa" {
        type primary;
        file "/etc/bind/db.192.168.17.signed"; 
    };
    ```
    
4. 重啟 BIND
    
    ```bash
    sudo systemctl restart bind9
    ```
    
5. **測試**
    
    ```bash
    dig @192.168.17.53 dnskey 17.nasa +dnssec
    dig @192.168.17.53 dnskey 17.168.192.in-addr.arpa +dnssec
    ```
    
    如果能看到 `DNSKEY`、`RRSIG (DNSKEY)`，表示簽名成功。
    
6. 輸出 DS Record，提交給 parent zone (OJ)
    
    ```bash
    sudo dnssec-dsfromkey -2 /etc/bind/K17.nasa.+013+41875.key 
    sudo dnssec-dsfromkey -2 /etc/bind/K17.168.192.in-addr.arpa.+013+15389.key
    ```
    
    Output:
    
    ```
    17.nasa. IN DS 41875 13 2 CDE016E19EB100A7EC76F3F0E4B760505E93DD3CBDFB73CB6E30867A003F8439
    17.168.192.in-addr.arpa. IN DS 15389 13 2 11877F6704C22191A791298241412DE9B297CDA03A1545477BE6D6BCCD9042DD
    ```
    
    Submit:
    
    ```
    41875 13 2 CDE016E19EB100A7EC76F3F0E4B760505E93DD3CBDFB73CB6E30867A003F8439
    15389 13 2 11877F6704C22191A791298241412DE9B297CDA03A1545477BE6D6BCCD9042DD
    ```
    

## Resolver

- 安裝 bind
    
    ```bash
    sudo apt install bind9
    sudo systemctl enable named
    ```
    
    - 把 judge 的 cert 和 key 讀進去，設定好權限
    
    ```bash
    sudo mkdir -p /etc/bind/certs
    sudo cp dns.17.nasa.crt /etc/bind/certs/
    sudo cp dns.17.nasa.key /etc/bind/certs/
    sudo chown root:bind /etc/bind/certs/dns.17.nasa.*
    sudo chmod 640 /etc/bind/certs/dns.17.nasa.*
    ```
    
- `named.conf.local`
    
    ```
    zone "17.nasa" {
            type forward;
            forward only;
            forwarders { 192.168.17.53; };
    };
    
    zone "17.168.192.in-addr.arpa" {
            type forward;
            forward only;
            forwarders { 192.168.17.53; };
    };
    
    zone "254.nasa" {
            type forward;
            forward only;
            forwarders { 192.168.254.153; };
    };
    
    zone "254.168.192.in-addr.arpa" {
            type forward;
            forward only;
            forwarders { 192.168.254.153; };
    };
    
    zone "nasa." {
            type forward;
            forward only;
            forwarders { 192.168.254.3; };
    };
    
    zone "168.192.in-addr.arpa" {
            type forward;
            forward only;
            forwarders { 192.168.254.3; };
    };
    ```
    
- 拿取 `nasa.` 和 `168.192.in-addr.arpa` 的 DNSKEY，加入 trust anchor
    
    ```bash
    dig @192.168.254.3 DNSKEY nasa.
    dig @192.168.254.3 DNSKEY 168.192.in-addr.arpa
    ```
    
- `named.conf.options`
    
    ```
    options {
            directory "/var/cache/bind";
    
            recursion yes;
            allow-recursion { any; };
    
            dnssec-validation yes;
    
            listen-on port 53 { any; };
            listen-on port 443 tls local-tls http local-http { any; };
            listen-on port 853 tls local-tls { any; };
    
    };
    
    tls local-tls {
            cert-file "/etc/bind/certs/dns.17.nasa.crt";
            key-file  "/etc/bind/certs/dns.17.nasa.key";
    };
    
    http local-http {
            endpoints { "/dns-query"; };
    };
    
    trust-anchors {
            "168.192.in-addr.arpa" static-key 257 3 13 "Gm+mztfbAQOMBL+ku0NuHT6ZPDhDn3trsXgMykfvtIAAZNX5unWY4qcb oRRAkZVc+Bfgk1ZnGi67b/SkiYbJfQ==";
            "nasa." static-key 257 3 13 "xlZIu5WnOyGXpnySEzBQLZoecNq4e0rPVQHSHUkn6UtpKuJPkVMmFlFI c/ex6hvim5bT/7u7UHbjPJnENzzgPA==";
    };
    ```
    
- 檢查 `named` 是否有在 port 53, 443, 853 上監聽
    
    ```bash
    sudo ss -tlnp | grep named
    ```
    
- 重開 named 服務
    
    ```bash
    sudo systemctl restart named
    ```
    

## Debug

- 可以直接觀察 iptables
    
    ```bash
    sudo iptables -L -v -n --line-numbers
    ```
    
- 插入 LOG rule 觀察被擋掉的封包細節
    
    ```bash
    sudo iptables -I FORWARD X -i enp0s3 -d 192.168.17.0/24 -j LOG --log-prefix "[enp0s3->LAN DROP] " --log-level 4
    sudo iptables -I FORWARD X -i wg0 -d 192.168.17.0/24 -j LOG --log-prefix "[wg0->LAN DROP] " --log-level 4
    ```
    
    `X` 為對應 DROP rule 的號碼，可以用
    
- 開啟 journal 觀查
    
    ```bash
    sudo journalctl -k -f
    ```
    

## Reference
- https://www.isc.org/blogs/bind-implements-doh-2021/
- https://downloads.isc.org/isc/bind9/9.18.35/doc/arm/html/index.html