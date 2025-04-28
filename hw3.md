# NA HW3

## Table of Content
* [Router](#router)
  * [DHCP](#dhcp)
  * [Firewall](#firewall)
* [DNS](#dns)
* [Mail](#mail)
  * [Update Time](#update-time)
  * [Certificates](#certificates)
  * [Install Required Packages](#install-required-packages)
  * [User Account](#user-account)
* [Postfix](#postfix)
  * [/etc/postfix/main.cf](#etcpostfixmaincf)
  * [/etc/postfix/master.cf](#etcpostfixmastercf)
  * [/etc/postfix/maps](#etcpostfixmaps)
  * [maps/outgoing_header_checks](#mapsoutgoing_header_checks)
  * [maps/sender_access](#mapssender_access)
  * [maps/sender_canonical](#mapssender_canonical)
  * [maps/sender_login](#mapssender_login)
  * [maps/virtual_alias_hash](#mapsvirtual_alias_hash)
  * [maps/virtual_alias_regexp](#mapsvirtual_alias_regexp)
* [Dovecot](#dovecot)
  * [/etc/dovecot/dovecot.conf](#etcdovecotdovecotconf)
  * [/etc/dovecot/conf.d/10-mail.conf](#etcdovecotconfd10-mailconf)
  * [/etc/dovecot/conf.d/10-auth.conf](#etcdovecotconfd10-authconf)
  * [/etc/dovecot/conf.d/10-master.conf](#etcdovecotconfd10-masterconf)
  * [/etc/dovecot/conf.d/10-ssl.conf](#etcdovecotconfd10-sslconf)
* [Postgrey](#postgrey)
  * [/etc/default/postgrey](#etcdefaultpostgrey)
* [Rspamd](#rspamd)
  * [local.d](#locald)
  * [actions.conf](#actionsconf)
  * [dkim_signing.conf](#dkim_signingconf)
  * [milter_headers.conf](#milter_headersconf)
  * [multimap.conf](#multimapconf)
  * [options.inc](#optionsinc)
  * [worker-controller.inc](#worker-controllerinc)
  * [worker-proxy.inc](#worker-proxyinc)
* [Test &amp; Debug](#test--debug)
  * [General](#general)
  * [Open relay test](#open-relay-test)
  * [Email spoofing test](#email-spoofing-test)
  * [Greylisting test](#greylisting-test)
  * [Alias &amp; Check Domain](#alias--check-domain)
  * [Sender rewrite](#sender-rewrite)
  * [SPAM](#spam)

## Router

### DHCP

- 將上次設定的 resolver 設定到 DHCP 的 name server 中
- Assign IP 給 Mail server，將 `/etc/dhcp/dhcpd.conf` 改成:
    
    ```bash
    subnet 192.168.${ID}.0 netmask 255.255.255.0 {
        range 192.168.${ID}.111 192.168.${ID}.222;
        option routers 192.168.${ID}.254;
        option domain-name-servers 192.168.${ID}.153, 8.8.8.8, 8.8.4.4; # Add resovler
        default-lease-time 600;
        max-lease-time 7200;
    
        host agent {
            hardware ethernet 08:00:27:44:92:EA; # Agent VM MAC
            fixed-address 192.168.${ID}.234;
        }
    
    		# NEW: add Mail server IP assignment
    		host mail {
    				hardware ethernet 08:00:27:75:62:85;
    				fixed-address 192.168.${ID}.25;
    		}
    		
        host authoritative-dns {
            hardware ethernet 08:00:27:D0:53:5E; # Authoritative DNS Server VM MAC
            fixed-address 192.168.${ID}.53; 
        }
    
        host resolver {
            hardware ethernet 08:00:27:38:6D:B6; # Resolver VM MAC 
            fixed-address 192.168.${ID}.153;
        }
    }
    ```
    
- 重啟 DHCP server
    
    ```bash
    sudo systemctl restart isc-dhcp-server
    ```
    

### Firewall

```bash
# 25  SMTP
sudo iptables -I FORWARD -p tcp --dport 25  -j ACCEPT
sudo iptables -I FORWARD  -p tcp --sport 25  -j ACCEPT

# 587 Submission
sudo iptables -I FORWARD -p tcp --dport 587 -j ACCEPT
sudo iptables -I FORWARD -p tcp --sport 587 -j ACCEPT

# 465 SMTPS（TLS‑wrapper）
sudo iptables -I FORWARD -p tcp --dport 465 -j ACCEPT
sudo iptables -I FORWARD -p tcp --sport 465 -j ACCEPT

# 143/993  IMAP & IMAPS
sudo iptables -I FORWARD -p tcp --dport 143 -j ACCEPT
sudo iptables -I FORWARD -p tcp --dport 993 -j ACCEPT

# Save & Reload
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

---

## DNS

> 可以把 dkim key 放在另一個檔案 include 進來
> 

| Name | Type | Content |
| --- | --- | --- |
| @ | **MX** | `mail.${ID}.nasa.` |
| @ | **TXT (SPF)** | `"v=spf1 mx a:mail.${ID}.nasa ip4:192.168.${ID}.25 ~all"` |
| mail | **MX** | `mail.${ID}.nasa.` |
| mail | **A** | `192.168.${ID}.25` |
| _dmarc | **TXT (DMARC)** | `"v=DMARC1; p=reject; adkim=s; aspf=s; rua=mailto:dmarc-report-rua@${ID}.nasa;"` |
| dkim._domainkey | **TXT (DKIM)** | `"v=DKIM1; k=rsa; p=…"` |

---

## Mail

### Update Time

```bash
timedatectl set-timezone Asia/Taipei
sudo apt update
```

### **Certificates**

從 OJ 下載 certificates 和 key，放在以下位置

- `/etc/ssl/certs/mail.${ID}.nasa.crt`
- `/etc/ssl/private/mail.${ID}.nasa.key`

設定權限

```bash
sudo chmod 600 /etc/ssl/private/mail.${ID}.nasa.key
```

### **Install Required Packages**

- **Postfix & Dovecot**
    
    > Postfix 安裝選擇 Internet Site，Postfix Mail Name 請填 `${ID}.nasa`。
    > 
    
    ```bash
    sudo apt update
    sudo apt install postfix dovecot-core dovecot-imapd dovecot-pop3d dovecot-lmtpd dovecot-sieve dovecot-managesieved sasl2-bin libsasl2-modules opendkim opendkim-tools postgrey
    ```
    
- **Rspamd**
    
    ```bash
    sudo apt-get install -y lsb-release wget gpg
    CODENAME=`lsb_release -c -s`
    sudo mkdir -p /etc/apt/keyrings
    wget -O- https://rspamd.com/apt-stable/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/rspamd.gpg > /dev/null
    echo "deb [signed-by=/etc/apt/keyrings/rspamd.gpg] http://rspamd.com/apt-stable/ $CODENAME main" | sudo tee /etc/apt/sources.list.d/rspamd.list
    echo "deb-src [signed-by=/etc/apt/keyrings/rspamd.gpg] http://rspamd.com/apt-stable/ $CODENAME main"  | sudo tee -a /etc/apt/sources.list.d/rspamd.list
    sudo apt-get update
    sudo apt-get --no-install-recommends install rspamd
    
    rspamadm configwizard
    sudo rspamd -v
    ```
    

### User Account

```bash
sudo adduser ta --allow-bad-names
sudo adduser cool-ta --allow-bad-names
```

在 `/etc/aliases`後加上

```bash
nasata: ta
```

---

## Postfix

### `/etc/postfix/main.cf`

> `${ID}` 改成自己的，其他不用，注意 restrictions 的先後順序會影響
> 

```bash
# --- Basic Identity ---
biff = no
dir = /etc/postfix/maps
myhostname = mail.${ID}.nasa
mydomain = ${ID}.nasa
myorigin = $mydomain
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain, mail.$mydomain
append_dot_mydomain = no

# --- Network Settings ---
inet_interfaces = all
inet_protocols = ipv4
mynetworks_style = host
relayhost =

# --- Mailbox Settings ---
mail_spool_directory = /var/mail/
mailbox_size_limit = 0
mailbox_command = /usr/lib/dovecot/deliver -d "$USER"

# --- SASL Authentication (via Dovecot) ---
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
broken_sasl_auth_clients = yes

# --- TLS/SSL Encryption ---
smtpd_tls_cert_file = /etc/ssl/certs/mail.${ID}.nasa.crt
smtpd_tls_key_file = /etc/ssl/private/mail.${ID}.nasa.key
smtpd_use_tls = yes
smtpd_tls_security_level = encrypt
smtpd_tls_loglevel = 1
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_security_level = may
smtp_tls_loglevel = 1
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

# --- Recipient Restrictions ---
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_auth_destination,
    permit_sasl_authenticated,
    check_policy_service unix:private/policyd-spf

smtpd_client_restrictions = 
    permit_sasl_authenticated, 
    check_policy_service inet:127.0.0.1:10023

# --- Sender Restrictions ---
smtpd_sender_login_maps = regexp:${dir}/sender_login
smtpd_sender_restrictions =
    reject_sender_login_mismatch,
    check_sender_access hash:${dir}/sender_access,
    permit_sasl_authenticated,
    reject_non_fqdn_sender

# --- Prevent Open Relay ---
smtpd_relay_restrictions = 
    permit_mynetworks, 
    permit_sasl_authenticated,
    reject_unauth_destination

# --- Milters (for DKIM, DMARC, Spam Filter) ---
smtpd_milters = inet:localhost:11332
non_smtpd_milters = $smtpd_milters
header_checks = regexp:/etc/postfix/maps/outgoing_header_checks

# --- Virtual Alias Maps ---
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
local_recipient_maps = proxy:unix:passwd.byname $alias_maps
virtual_alias_maps = hash:${dir}/virtual_alias_hash, regexp:${dir}/virtual_alias_regexp

# --- Canonical Maps (for sender rewriting) ---
sender_canonical_maps = hash:${dir}/sender_canonical
local_header_rewrite_clients = permit_sasl_authenticated
append_at_myorigin = yes
masquerade_domains = $mydomain
masquerade_classes = envelope_sender, header_sender, header_recipient
masquerade_exceptions = admin, root

# --- Other Settings ---
recipient_delimiter = +
message_size_limit = 51200000
mailbox_size_limit = 0
compatibility_level = 3.6

# --- Debug Settings ---
debug_peer_list = ${ID}.nasa, mail.${ID}.nasa, 127.0.0.1, ta.nasa, 10.113.0.0/16
debug_peer_level = 2
maillog_file=/var/log/mail.log
```

### **`/etc/postfix/master.cf`**

```bash
submission     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/submissions
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_sasl_type=dovecot
  -o smtpd_sasl_path=private/auth
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject

policyd-spf    unix  -       n       n       -       0       spawn
  user=policyd-spf argv=/usr/bin/policyd-spf
  
127.0.0.1:10025 inet n - - - - smtpd
  -o content_filter=
  -o local_recipient_maps=
  -o relay_recipient_maps=
  -o smtpd_restriction_classes=
  -o smtpd_delay_reject=no
  -o smtpd_client_restrictions=permit_mynetworks,reject
  -o smtpd_helo_restrictions=
  -o smtpd_sender_restrictions=
  -o smtpd_recipient_restrictions=permit_mynetworks,reject
  -o mynetworks=127.0.0.0/8
  -o smtpd_authorized_xforward_hosts=127.0.0.0/8
  -o smtpd_tls_security_level=none
```

### `/etc/postfix/maps`

> 是 hash 檔的記得要用 `postmap` 產生 `.db` 檔
> 

```
maps
├── outgoing_header_checks
├── sender_access
├── sender_access.db
├── sender_canonical
├── sender_canonical.db
├── sender_login
├── virtual_alias_hash
├── virtual_alias_hash.db
└── virtual_alias_regexp
```

### `maps/outgoing_header_checks`

> `博士班` 要先 encode 成 base64 格式
> 

```bash
/^Subject:.*Graduate School/    REJECT Outgoing mail subject blocked.
/^Subject:.*5Y2a5aOr54\+t/      REJECT Outgoing mail subject blocked.
```

### `maps/sender_access`

```bash
<> REJECT       null sender
```

```bash
sudo postmap /etc/postfix/maps/sender_access
```

### `maps/sender_canonical`

> 記得要是大寫，不然 judge 會報錯
> 

```bash
cool-TA                 supercooool-TA
```

```bash
sudo postmap /etc/postfix/maps/sender_canonical
```

### `maps/sender_login`

```bash
/^cool-ta(\+[^@]+)?@(mail\.)?${ID}\.nasa$/  cool-ta
/^ta(\+[^@]+)?@(mail\.)?${ID}\.nasa$/       ta
/^([^+]+)(\+[^@]+)?@(mail\.)?${ID}\.nasa$/  $1
```

### `maps/virtual_alias_hash`

```bash
@mail.${ID}.nasa @${ID}.nasa
nasata ta
```

```bash
sudo postmap /etc/postfix/maps/virtual_alias_hash
```

### `maps/virtual_alias_regexp`

> `@mail.${ID}.nasa` → `@${ID}.nasa`
`<user>+<word>@` → `<user>@`
> 

```bash
/^([^+]+)(\+[^@]+)?@(mail\.)?${ID}\.nasa$/  $1@${ID}.nasa
```

---

## Dovecot

### `/etc/dovecot/dovecot.conf`

> 確定 `!include conf.d/*.conf` 沒備注解掉
> 

```
- listen = *, ::
+ listen = *, ::
```

### `/etc/dovecot/conf.d/10-mail.conf`

> 設定 Mail 路徑，要跟 Poster 使用相同格式
> 

```
- mail_location = mbox:~/mail:INBOX=/var/mail/%u
+ mail_location = maildir:~/Maildir
```

### `/etc/dovecot/conf.d/10-auth.conf`

> 認證管理，確定 `disable_plaintext_auth = yes` 沒被註解
> 

```
- auth_mechanisms = plain
+ auth_mechanisms = plain login

- auth_username_format = %Ln
+ auth_username_format = %Lu
```

### `/etc/dovecot/conf.d/10-master.conf`

```
service auth {
  unix_listener /var/spool/postfix/private/auth {
-   mode = 0666
+   mode = 0660
+   user = postfix
+   group = postfix
  }
}

service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
-   mode = 0666
+   mode = 0600
+   user = postfix
+   group = postfix
  }
}

service imap-login {
  inet_listener imap {
-   #port = 143
+		port = 143
  }
  inet_listener imaps {
-   #port = 993
-   #ssl = yes
+   port = 993
+   ssl = yes
  }
}
```

### `/etc/dovecot/conf.d/10-ssl.conf`

```
- ssl_cert = </etc/dovecot/private/dovecot.pem
- ssl_key = </etc/dovecot/private/dovecot.key
+ ssl_cert = </etc/ssl/certs/mail.${ID}.nasa.crt
+ ssl_key = </etc/ssl/private/mail.${ID}.nasa.key
```

---

## Postgrey

> 記得把 `ta@ta.nasa` 加到 `/etc/postgrey/whitelist_clients` 底下
> 

### `/etc/default/postgrey`

```
- POSTGREY_OPTS="--inet=10023"
+ POSTGREY_OPTS="--inet=127.0.0.1:10023 --delay=15"
```

```bash
sudo systemctl restart postgrey
```

---

## Rspamd

> 強烈建議使用，一個多合一的工具，可以同時設定 Spam filter、DKIM signing、Greylist, etc.
> 

### `local.d`

> `local.d` 的設定會覆蓋掉預設值，且每個檔案名稱是固定的，不能亂改
> 

```
/etc/rspamd/local.d
├── actions.conf
├── dkim_signing.conf
├── milter_headers.conf
├── module.conf.example
├── multimap.conf
├── options.inc
├── worker-controller.inc
└── worker-proxy.inc
```

全部設定完後，記得檢查有沒有語法錯誤

```bash
sudo rspamadm configtest
sudo systemctl restart rspamd
```

### `actions.conf`

> 注意 `%s` 前面沒有空格
> 

```bash
reject = null; # 避免擋掉 GTUBE
greylist = null;
add_header = 3;
rewrite_subject = 4;
grow_factor = 10;
subject = "**SPAM**%s"
```

### `dkim_signing.conf`

> 記得將產生的 public key file 複製到 Authoritative DNS
> 

```bash
sudo rspamadm dkim_keygen -s dkim -d ${ID}.nasa -k /var/lib/rspamd/dkim/${ID}.nasa.dkim.key | sudo tee /var/lib/rspamd/dkim/${ID}.nasa.dkim.txt > /dev/null
sudo chown _rspamd:_rspamd /var/lib/rspamd/dkim/${ID}.nasa.dkim.key
sudo chmod 600 /var/lib/rspamd/dkim/${ID}.nasa.dkim.key
```

```bash
enable = true;
path = "/var/lib/rspamd/dkim/${ID}.nasa.dkim.key";
selector = "dkim";
allow_hdrfrom_mismatch = true;
allow_username_mismatch = true;
use_esld = true;
use_domain = "envelope";
domain {
  ${ID}.nasa {
    path = "/var/lib/rspamd/dkim/${ID}.nasa.dkim.key";
    selector = "dkim";
  }
}
```

### `milter_headers.conf`

```bash
extended_spam_headers = true;
skip_local = false;
skip_authenticated = false;
use = ["x-spam-status", "x-spam-level"];
subject = "**SPAM**%s"
```

### `multimap.conf`

```bash
GTUBE_REWRITE {
  action = "rewrite subject";
  subject = "**SPAM**%s";
  score   = 0.0;
  type = "content";
  filter = "body"
  map  = "regexp;/etc/rspamd/maps/gtube.map";
  description = "Rewrite subject for GTUBE test";
}
```

- `/etc/rspamd/maps/gtube.map`
    
    ```bash
    /XJS\*C4JDBQADN1\.NSBN3\*2IDNEN\*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL\*C\.34X/
    ```
    

### `options.inc`

```bash
gtube_patterns = "disable";
```

### `worker-controller.inc`

```bash
password = "$2$yiqt8i9xi68yh3fne1si9joya7xozc64$enu7csuxos7pr4xghf8868t6f954td98be8dhhxu4jhiykn4g11b";
bind_socket = "0.0.0.0:11334";
```

### `worker-proxy.inc`

```bash
count = 2;
```

## Test & Debug

### General

- 更新設定完後記得重開
    
    ```bash
    sudo postfix reload
    sudo systemctl restart dovecot
    sudo systemctl restart rspamd
    sudo systemctl restart postgrey
    ```
    
- 確認信是否寄出 or 卡在 queue 裡（grep queueID 確認問題在哪）
    
    ```bash
    
    sudo mailq # or sudo postqueue -p
    sudo grep '{QueueID}' /var/log/mail.log
    sudo postqueue -f # flush queue
    ```
    
    - `status=sent`: 成功寄出
    - `status=deferred`: Temporary failure。可能原因：Connection timed out, Temporary lookup failure)，可能是防火牆擋住、網路問題等等
    - `status=bounced`: Permanent failure。找不到該 domain 的 IP，或是被收信 server 擋掉
    - `relay=none`: 不知道要寄給誰，可能是 DNS 問題
- 確認 mapping file 是否有語法錯誤
    
    ```bash
    sudo postmap -q "{QUERY}" {FORMAT}:maps/{FILE}
    sudo postmap -q "TA@${ID}.nasa" regexp:maps/sender_login # example
    > ta
    ```
    
- 看 log＆找關鍵字
    
    ```bash
    sudo tail -f /var/log/mail.log 
    sudo grep "{keyword}" /var/log/mail.log 
    
    sudo tail -f /var/log/rspamd/rspamd.log
    sudo grep "{keyword}" /var/log/rspamd/rspamd.log
    ```
    
- 確認每台機器是否可以拿到 DNS Record，否則需查看 resolver 設定，若更新完發現還拿不到，可能是 resolver 還留著 negative cahce。
    
    ```bash
    cat /etc/resolv.conf # on each server
    sudo rndc flush # on resolver
    ```
    

### Open relay test

```bash
swaks --server mail.${ID}.nasa --tls \
  --from checker@ta.nasa --to null@mail.ta.nasa
  --header "Subject: Open Relay Test" --body "Testing open relay."
> 554 5.7.1 Relay access denied
```

### Email spoofing test

```bash
swaks --auth LOGIN --auth-user TA --auth-password ${PASSWD} \
      --to user@gmail.com --from cool-TA@${ID}.nasa \
      --server mail.${ID}.nasa:587 --tls
> 553 5.7.1 Sender address rejected: not own by user ta

swaks --auth LOGIN --auth-user TA --auth-password E75F5C7165D5A632040C01CB09DC35B8\
      --to user@gmail.com --from TA@${ID}.nasa \
      --server mail.${ID}.nasa:587 --tls
> 250 2.0.0 Ok # 確認一下正常登入不能被擋掉
```

### Greylisting test

```bash
# First try -> reject
swaks --from any@gmail.com --to TA@${ID}.nasa --server mail.${ID}.nasa --tls
> 450 4.2.0 Client host rejected: Greylisted

# Second try -> passed 
swaks --from any@example.com --to TA@${ID}.nasa --server mail.${ID}.nasa --tls
> 250 2.0.0 Ok
```

### Alias & Check Domain

> 到 `/home/ta/Maildir/new` 底下查看是否有收到信
> 

```bash
# Test 1: NASATA@ -> TA@
swaks --auth LOGIN --auth-user TA --auth-password ${PASSWD} \
			--from TA@${ID}.nasa --to NASATA@${ID}.nasa \
      --server mail.${ID}.nasa:587 --tls \
      --header "Subject: Virtaul alias test" --body "hi"

# Test 2: <user>+<word>@ -> <user>@
swaks --auth LOGIN --auth-user TA --auth-password ${PASSWD} \
			--from TA@${ID}.nasa --to TA+abc@${ID}.nasa \
      --server mail.${ID}.nasa:587 --tls \
      --header "Subject: Virtaul alias test" --body "hi"
```

### Sender rewrite

> 要確認 Envolope 和 From: 都有改到
> 

```bash
# Test 1: @mail.${ID}.nasa -> @${ID}.nasa
swaks --auth LOGIN --auth-user TA --auth-password ${PASSWD} \
			--from TA@mail.${ID}.nasa --to ta@${ID}.nasa \
      --server mail.${ID}.nasa:587 --tls \
      --header "Subject: Sender Rewrite Test1" --body "早安"

# Test 2: cool-TA@ -> supercooool-TA@
swaks --auth LOGIN --auth-user cool-TA --auth-password ${PASSWD} \
			--from cool-TA@${ID}.nasa --to ta@${ID}.nasa \
      --server mail.${ID}.nasa:587 --tls \
      --header "Subject: Sender Rewrite Test2" --body "早安"
```

### SPAM

> 要注意某些信會被直接 reject 掉
> 

```bash
cat > /tmp/gtube.eml <<'EOF'
From: ta@ta.nasa
To: ta@${ID}.nasa
Subject: GTUBE test

This is the GTUBE, the
Generic
Test for
Unsolicited
Bulk
Email

If your spam filter supports it, the GTUBE provides a test by which you
can verify that the filter is installed correctly and is detecting incoming
spam. You can send yourself a test mail containing the following string of
characters (in upper case and with no white spaces and line breaks):

XJS*C4JDBQADN1.NSBN3*2IDNEN*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL*C.34X
EOF

swaks --auth LOGIN --auth-user TA --auth-password ${PASSWD} \
			--from ta@${ID}.nasa --to ta@${ID}.nasa  \
			--server mail.${ID}.nasa:587 --tls \
			--data @/tmp/gtube.eml
> 250 2.0.0 Ok # 不能被 reject，否則 OJ 會報錯
```

直接確認 rspamd 的行為

```
rspamc -h 127.0.0.1:11333 /tmp/gtube.eml 

Results for file: /tmp/gtube.eml (4.16 seconds)
[Metric: default]
Action: rewrite subject
Spam: true
Subject: **SPAM**GTUBE test
Score: -0.10 / 4.00
Symbol: ARC_NA (0.00)
Symbol: DMARC_NA (0.00)[ta.nasa]
Symbol: FROM_NO_DN (0.00)
Symbol: GTUBE_REWRITE (0.00)
Symbol: MIME_GOOD (-0.10)[text/plain]
Symbol: MIME_TRACE (0.00)[0:+]
Symbol: R_DKIM_NA (0.00)
Message-ID: undef
Message - smtp_message: Matched map: GTUBE_REWRITE
```