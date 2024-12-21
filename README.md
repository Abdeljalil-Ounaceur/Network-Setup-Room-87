# Using XUbuntu_24 LTS

# Project GOAL: Installing Various Network Services for Mini Enterprise
Including DNS, AUTHENTICATION, MAIL, WEB, SHARED STORAGE and SECURITY

## 1. Configuring a Static IP Address

### Required Tools: "network-manager"
```bash
apt install network-manager
```

### Show Connections and Modify
Show connections and modify the one that you are connected to (or just 'modify Wired connection 1' if you are connected by a cable)
```bash
nmcli conn show
nmcli conn mod 'Wired connection 1' ipv4.method manual ipv4.addresses "192.168.1.200/24" ipv4.gateway "192.168.1.1" ipv4.dns "8.8.8.8,8.8.4.4" 
nmcli conn up 'Wired connection 1'
```

### Check IP and Test Connectivity
```bash
ip a
ping google.com
```

## 2. Configuring DNS

### Required Tools
```bash
apt install bind9
apt install dnsutils  # recommended
```

### Configure named.conf.options
Edit `named.conf.options`:
```bash
nano named.conf.options
```

Uncomment forwarders and add:
```
forwarders {
    8.8.8.8;
    8.8.4.4;
};
```

### Create Zone Files
```bash
cd /etc/bind
cp db.local db.groupe1.master2.fsa.ma.zone
```

Edit the zone file:
```bash
nano db.groupe1.master2.fsa.ma.zone
```

Change any appearance of 'localhost' to 'groupe1.master2.fsa.ma' and change 127.0.0.1 to your static IP address:
```
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     groupe1.master2.fsa.ma. root.groupe1.master2.fsa.ma. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      groupe1.master2.fsa.ma.
@       IN      A       192.168.1.200
@       IN      AAAA    ::1
```

### Create Reverse Zone File
```bash
cp db.127 db.192
nano db.192
```

Modify it to:
```
;
; BIND reverse data file for local 192.168.1.XXX interface
;
$TTL    604800
@       IN      SOA     groupe1.master2.fsa.ma. root.groupe1.master2.fsa.ma. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      groupe1.master2.fsa.ma.
200     IN      PTR     groupe1.master2.fsa.ma.
```

### Modify named.conf.local
Add:
```
zone "groupe1.master2.fsa.ma" {
        type master;
        file "/etc/bind/db.groupe1.master2.fsa.ma.zone";
};

zone "1.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/db.192";
};
```

### Restart and Configure Network
```bash
systemctl restart bind9
nmcli conn mod 'Wired connection 1' ipv4.dns "127.0.0.1"
nmcli conn up 'Wired connection 1'
```

### Test Configuration
```bash
ping groupe1.master2.fsa.ma
nslookup groupe1.master2.fsa.ma localhost
dig -x 192.168.1.200
ping google.com
```

## 3. Setup LDAP for User Centralization

### Required Tools
```bash
apt install slapd ldap-utils
```
When prompted to enter a password, just press enter.

### Reconfigure LDAP
```bash
dpkg-reconfigure slapd
```

Enter in order:
- No
- groupe1.master2.fsa.ma
- Groupe1 Master2 FSA
- some_password
- some_password (again)
- yes
- yes

### Configure LDAP
Edit `/etc/ldap/ldap.conf`:
```bash
nano /etc/ldap/ldap.conf
```

Uncomment and edit BASE and URI:
```
BASE    dc=groupe1,dc=master2,dc=fsa,dc=ma
URI     ldap://ldap.groupe1.master2.fsa.ma
```

### Add DNS Entries
Add to `/etc/bind/db.groupe1.master2.fsa.ma.zone`:
```
ldap    IN      A       192.168.1.200
```

Add to `/etc/bind/db.192`:
```
200     IN      PTR     ldap.groupe1.master2.fsa.ma.
```

### Restart and Test
```bash
systemctl restart bind9
ping ldap.groupe1.master2.fsa.ma
ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config cn
ldapsearch -x -H ldap://ldap.groupe1.master2.fsa.ma -b dc=groupe1,dc=master2,dc=fsa,dc=ma
```

### Base Tree Structure
```
dc=groupe1,dc=master2,dc=fsa,dc=ma
├── ou=Users
│   ├── uid=john_doe
│   ├── uid=admin
├── ou=Groups
│   ├── cn=Students
│   ├── cn=Admins
│   ├── cn=Teachers
│   ├── cn=Developers
├── ou=Computers
│   ├── cn=workstation1
│   ├── cn=server1
├── ou=Services
│   ├── cn=Mail
│   ├── cn=VPN
│   ├── cn=Samba
├── ou=Rooms
│   ├── cn=Room87
│   ├── cn=Room88
├── ou=Applications
├── ou=Policies
└── ou=Publications
```

### Create Base Tree LDIF
Create `base_tree.ldif`:
```ldif
dn: ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: organizationalUnit
ou: Users

dn: ou=Groups,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: organizationalUnit
ou: Groups

dn: ou=Computers,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: organizationalUnit
ou: Computers

dn: ou=Services,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: organizationalUnit
ou: Services

dn: ou=Rooms,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: organizationalUnit
ou: Rooms

dn: ou=Applications,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: organizationalUnit
ou: Applications

dn: ou=Policies,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: organizationalUnit
ou: Policies

dn: ou=Publications,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: organizationalUnit
ou: Publications
```

Add to database:
```bash
ldapadd -x -D "cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -f base_tree.ldif
```

### Add Users
Create `add_users.ldif`:
```ldif
dn: uid=john_doe,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: john_doe
cn: John Doe
sn: Doe
mail: john_doe@groupe1.master2.fsa.ma
uidNumber: 10001
gidNumber: 20000
homeDirectory: /home/john_doe
loginShell: /bin/bash
userPassword: {SSHA}U2HQDMA9QTCb5CA5z3F846ZnSy0EXtJN

dn: uid=jane_doe,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: jane_doe
cn: Jane Doe
sn: Doe
mail: jane_doe@groupe1.master2.fsa.ma
uidNumber: 10002
gidNumber: 20000
homeDirectory: /home/jane_doe
loginShell: /bin/bash
userPassword: {SSHA}U2HQDMA9QTCb5CA5z3F846ZnSy0EXtJN

dn: uid=admin1,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: admin1
uid: admin1
mail: admin1@groupe1.master2.fsa.ma
uidNumber: 10003
gidNumber: 20001
homeDirectory: /home/admin1
loginShell: /bin/bash
userPassword: {SSHA}U2HQDMA9QTCb5CA5z3F846ZnSy0EXtJN
```

### Add Groups
Create `add_groups.ldif`:
```ldif
dn: cn=Students,ou=Groups,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: posixGroup
cn: Students
gidNumber: 20000
memberUid: john_doe
memberUid: jane_doe

dn: cn=Admins,ou=Groups,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: posixGroup
cn: Admins
gidNumber: 20001
memberUid: admin1
```

### Add Services
Create `add_services.ldif`:
```ldif
dn: cn=Mail,ou=Services,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: organizationalRole
cn: Mail
description: Central Mail Service

dn: cn=VPN,ou=Services,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: organizationalRole
cn: VPN
description: VPN Service

dn: cn=Samba,ou=Services,dc=groupe1,dc=master2,dc=fsa,dc=ma
objectClass: organizationalRole
cn: Samba
description: File Sharing Service
```

### Add Data to Database
```bash
ldapadd -x -D "cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -f add_users.ldif
ldapadd -x -D "cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -f add_groups.ldif
ldapadd -x -D "cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -f add_services.ldif
```

### Install and Configure ldapscripts
```bash
apt install ldapscripts
```

Edit `/etc/ldapscripts/ldapscripts.conf`:
```
SERVER="ldap://ldap.groupe1.master2.fsa.ma"

SUFFIX="dc=groupe1,dc=master2,dc=fsa,dc=ma"
GSUFFIX="ou=Groups"   
USUFFIX="ou=Users"    
MSUFFIX="ou=Computers"

BINDDN="cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma"
BINDPWDFILE="/etc/ldapscripts/ldapscripts.passwd"

GIDSTART="20000" 
UIDSTART="10000" 
MIDSTART="30000"
```

Store password:
```bash
echo -n 'password' | sudo tee /etc/ldapscripts/ldapscripts.passwd
chmod 400 /etc/ldapscripts/ldapscripts.passwd
```

### Create User Template
Edit `/etc/ldapscripts/ldapscripts.conf`:
```
UTEMPLATE="/etc/ldapscripts/ldapadduser.template"
```

Create `/etc/ldapscripts/ldapadduser.template`:
```ldif
dn: uid=<user>,<usuffix>,<suffix>
objectClass: account
objectClass: posixAccount
objectclass: postfixUser
objectclass: top
cn: <user>
uid: <user>
uidNumber: <uid>
gidNumber: <gid>
homeDirectory: /home/mail/<user>
mailacceptinggeneralid: <user>@groupe1.master2.fsa.ma
mailacceptinggeneralid: <user>@mail.groupe1.master2.fsa.ma
loginShell: <shell>
gecos: <user>
description: User account
```

### Configure Access Control List
Create `ACL.ldif`:
```ldif
changetype: modify
replace: olcAccess
olcAccess: to attrs=userPassword
  by self write
  by group.exact="cn=Admins,ou=Groups,dc=groupe1,dc=master2,dc=fsa,dc=ma" write
  by anonymous auth
  by * none
olcAccess: to dn.regex="uid=([^,]+),ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma"
  by dn.exact,expand="uid=$1,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma" write
  by group.exact="cn=Admins,ou=Groups,dc=groupe1,dc=master2,dc=fsa,dc=ma" write
  by * none
olcAccess: to dn.subtree="ou=Groups,dc=groupe1,dc=master2,dc=fsa,dc=ma"
  by group.exact="cn=Admins,ou=Groups,dc=groupe1,dc=master2,dc=fsa,dc=ma" write
  by * read
olcAccess: to dn.exact="cn=Samba,ou=Services,dc=groupe1,dc=master2,dc=fsa,dc=ma"
  by group.exact="cn=Students,ou=Groups,dc=groupe1,dc=master2,dc=fsa,dc=ma" read
  by * none
olcAccess: to dn.exact="cn=VPN,ou=Services,dc=groupe1,dc=master2,dc=fsa,dc=ma"
  by group.exact="cn=Admins,ou=Groups,dc=groupe1,dc=master2,dc=fsa,dc=ma" read
  by * none
olcAccess: to *
  by group.exact="cn=Admins,ou=Groups,dc=groupe1,dc=master2,dc=fsa,dc=ma" write
  by self write
  by * read
```

Apply ACL:
```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f ACL.ldif
```

Test changes:
```bash
ldapsearch -x -D "uid=john_doe,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -b "uid=john_doe,ou=Users,dc=groupe1,dc

# LDAP, Postfix, and Dovecot Configuration Guide

## LDAP Security Configuration

### Check User Restrictions
To verify that users cannot see other users' data (should display '32 No such Object'):
```bash
ldapsearch -x -D "uid=john_doe,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -b "uid=jane_doe,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma"
```

### Check Anonymous User Access
To verify anonymous users can only see general data but not sensitive data:
```bash
ldapmodify -x -D "uid=admin,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W
```

### Securing LDAP with TLS

#### Create TLS Certificates
```bash
openssl genrsa -out CA.key 8192 && chmod 400 CA.key
openssl req -new -x509 -nodes -key CA.key -days 3650 -out CA.pem
openssl genrsa -out device.key 4096 && chmod 400 device.key
openssl req -new -key device.key -out device.csr
openssl x509 -req -in device.csr -CA CA.pem -CAkey CA.key -CAcreateserial -out device.crt -days 365
```

#### Save Certificates Securely
```bash
mkdir /etc/ldap/tls/
cp CA.pem device.key device.crt /etc/ldap/tls/
chown -R openldap:openldap /etc/ldap/tls/
chmod 101 /etc/ldap/tls/
chmod 400 /etc/ldap/tls/*
chmod 404 /etc/ldap/tls/CA.pem
```

#### Enable TLS Configuration
Create file `TLS_enable.ldif`:
```ldif
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ldap/tls/CA.pem
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ldap/tls/device.crt
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/tls/device.key
```

Activate TLS:
```bash
ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f TLS_enable.ldif
```

Edit `/etc/ldap/ldap.conf`:
```
TLS_CACERT /etc/ldap/tls/CA.pem
```

Test TLS connection:
```bash
ldapsearch -LLL -x -H ldap://ldap.example.com -b dc=ldap,dc=example,dc=com -ZZ
```

## Postfix Configuration

### Installation
```bash
apt install postfix postfix-ldap
apt install mailutils
```

When prompted:
- Select "Internet site" as initial type of configuration
- Enter mail.groupe1.master2.fsa.ma as your mail host

### DNS Configuration
Add to `/etc/bind/db.groupe1/master2.fsa.ma.zone`:
```
mail    IN    A    192.168.1.200
@       IN    MX   10 mail.groupe1.master2.fsa.ma.
```

Add to `/etc/bind/db.192`:
```
200    IN    PTR    mail.groupe1.master2.fsa.ma.
```

Restart BIND:
```bash
systemctl restart bind9
```

### LDAP Schema Configuration
```bash
wget -O postfix.ldif https://raw.githubusercontent.com/68b32/postfix-ldap-schema/master/postfix.ldif
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f postfix.ldif
```

### Test User Setup
```bash
ldapdeleteuser john_doe
ldapadduser john_doe Students
ldappasswd john_doe
```

### Postfix LDAP Configuration Files

Create file `virtual_alias_domains`:
```
server_host = ldap://ldap.groupe1.master2.fsa.ma
start_tls = yes
version = 3
tls_ca_cert_file = /etc/ldap/tls/CA.pem
tls_require_cert = yes

bind = yes
bind_dn = cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma
bind_pw = toor

search_base = ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma
scope = sub

query_filter = mailacceptinggeneralid=*@%s
result_attribute = mailacceptinggeneralid
result_format = %d
```

Create file `virtual_alias_maps`:
```
server_host = ldap://ldap.groupe1.master2.fsa.ma
start_tls = yes
version = 3
tls_ca_cert_file = /etc/ldap/tls/CA.pem
tls_require_cert = yes

bind = yes
bind_dn = cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma
bind_pw = toor

search_base = ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma
scope = sub

query_filter = mailacceptinggeneralid=%s
result_attribute = maildrop
```

Create file `virtual_mailbox_maps`:
```
server_host = ldap://ldap.groupe1.master2.fsa.ma
start_tls = yes
version = 3
tls_ca_cert_file = /etc/ldap/tls/CA.pem
tls_require_cert = yes

bind = yes
bind_dn = cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma
bind_pw = toor

search_base = ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma
scope = sub

query_filter = maildrop=%s
result_attribute = homeDirectory
result_format = %s/mailbox/
```

Create file `virtual_uid_maps`:
```
server_host = ldap://ldap.groupe1.master2.fsa.ma
start_tls = yes
version = 3
tls_ca_cert_file = /etc/ldap/tls/CA.pem
tls_require_cert = yes

bind = yes
bind_dn = cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma
bind_pw = toor

search_base = ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma
scope = sub

query_filter = maildrop=%s
result_attribute = uidNumber
```

Create file `smtpd_sender_login_maps`:
```
server_host = ldap://ldap.groupe1.master2.fsa.ma
start_tls = yes
version = 3
tls_ca_cert_file = /etc/ldap/tls/CA.pem
tls_require_cert = yes

bind = yes
bind_dn = cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma
bind_pw = toor

search_base = ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma
scope = sub

query_filter = (|(mailacceptinggeneralid=%s)(maildrop=%s))
result_attribute = uid
```

### Deploy Configuration Files
```bash
mkdir /etc/postfix/ldap
cp virtual* smtpd_sender_login_maps /etc/postfix/ldap/
```

### Test Configuration
```bash
postmap -q john_doe@groupe1.master2.fsa.ma ldap:/etc/postfix/ldap/virtual_mailbox_maps
postmap -q john_doe@groupe1.master2.fsa.ma ldap:/etc/postfix/ldap/virtual_uid_maps
postmap -q john_doe@groupe1.master2.fsa.ma ldap:/etc/postfix/ldap/virtual_alias_maps
```

### Secure Configuration Files
```bash
chown postfix:postfix /etc/postfix/ldap/*
chmod 400 /etc/postfix/ldap/*
```

### TLS Configuration
```bash
mkdir -p /var/spool/postfix/etc/ldap/tls
touch /var/spool/postfix/etc/ldap/tls/CA.pem
mount --bind /etc/ldap/tls/CA.pem /var/spool/postfix/etc/ldap/tls/CA.pem
```

Create file `/etc/systemd/system/var-spool-postfix-etc-ldap-tls-CA.pem.mount`:
```ini
[Unit]
Description=Bind /etc/ldap/tls/CA.pem to /var/spool/postfix/etc/ldap/tls/CA.pem
Before=postfix.service

[Mount]
What=/etc/ldap/tls/CA.pem
Where=/var/spool/postfix/etc/ldap/tls/CA.pem
Type=none
Options=bind

[Install]
WantedBy=postfix.service
```

Enable TLS mount:
```bash
systemctl daemon-reload
systemctl start var-spool-postfix-etc-ldap-tls-CA.pem.mount
mount | grep CA
systemctl enable var-spool-postfix-etc-ldap-tls-CA.pem.mount
```

### Configure Postfix Main Configuration
Edit `/etc/postfix/main.cf`:
```
myhostname = mail.groupe&.master2.fsa.ma

virtual_alias_domains = ldap:/etc/postfix/ldap/virtual_alias_domains
virtual_mailbox_domains = $myhostname

virtual_alias_maps = ldap:/etc/postfix/ldap/virtual_alias_maps
virtual_mailbox_base = /
virtual_mailbox_maps = ldap:/etc/postfix/ldap/virtual_mailbox_maps
virtual_uid_maps = ldap:/etc/postfix/ldap/virtual_uid_maps
virtual_gid_maps = ldap:/etc/postfix/ldap/virtual_uid_maps

smtpd_sender_login_maps = ldap:/etc/postfix/ldap/smtpd_sender_login_maps
```

### Create Mail Directory
```bash
mkdir -p /home/mail
chmod o+w /home/mail
```

### Reload Postfix
```bash
systemctl reload postfix
```

## Dovecot Configuration

### Installation
```bash
apt install dovecot-core dovecot-imapd dovecot-ldap
```

### Basic Configuration
Edit `/etc/dovecot/conf.d/10-mail.conf`:
```
maildir:~/mailbox
```

Edit `/etc/dovecot/conf.d/10-master.conf`:
```
service imap-login {
    inet_listener imap {
        port = 0
    }
    inet_listener imaps {
        port = 993
        ssl = yes
    }
    
    service_count = 1
    process_min_avail = 1
}
```

### Authentication Configuration
Edit `/etc/dovecot/conf.d/10-auth.conf`:
```
#!include auth-system.conf.ext
!include auth-ldap.conf.ext
```

Edit `/etc/dovecot/conf.d/auth-ldap.conf.ext`:
```
passdb {
        driver = ldap
        args = /etc/dovecot/dovecot-ldap.conf.ext
}
userdb {
        driver = ldap
        args = /etc/dovecot/dovecot-ldap.conf.ext
}
```

Edit `/etc/dovecot/dovecot-ldap.conf.ext`:
```
uris = ldap://ldap.groupe1.master2.fsa.ma
dn = cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma
dnpass = your_admin_password
tls = yes
tls_ca_cert_file = /etc/ldap/tls/CA.pem
tls_require_cert = hard
debug_level = 0
auth_bind = yes
auth_bind_userdn = uid=%u,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma
ldap_version = 3
base = ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma
scope = subtree
user_attrs = homeDirectory=home,uidNumber=uid,gidNumber=gid
user_filter = (&(objectClass=posixAccount)(uid=%u))
```

### SSL/TLS Configuration
Edit `/etc/dovecot/conf.d/10-auth.conf`:
```
ssl = required
ssl_cert = </etc/dovecot/tls/server.crt
ssl_key = </etc/dovecot/tls/server.key
ssl_dh_parameters_length = 4096
ssl_protocols = !SSLv2 !SSLv3
ssl_cipher_list = ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA
ssl_prefer_server_ciphers = yes
verbose_ssl = yes
```

### Configure Authentication Socket
Edit `/etc/dovecot/conf.d/10-master.conf`:
```
service auth {
    unix_listener /var/spool/postfix/private/auth {
        mode = 0660
        user = postfix
        group = postfix
    }
}
```

### Configure Postfix SASL
Edit `/etc/postfix/main.cf`:
```
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
broken_sasl_auth_clients = yes
```



### Test Authentication:
```
openssl s_client -starttls smtp -connect mail.groupe1.master2.fsa.ma:25
```
