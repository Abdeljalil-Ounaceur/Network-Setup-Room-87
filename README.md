# Configuration Réseau Salle 87
_Utilisation de XUbuntu_24 LTS_  
_Sous-réseau 10.2.0.0/16_  
_Utilisé l’interface 'Wired connection 1'_  

# Objectif
Ce document de synthèse présente la configuration complète de plusieurs services réseau pour la salle 87. Dans l’objectif d’offrir un réseau fiable et sécurisé, différentes briques logicielles ont été mises en place pour gérer l’adressage IP, la résolution de noms, l’authentification centralisée, la messagerie, le partage de ressources et la sécurité globale du système.

1.  Configuration IP Statique
Le choix d’une adresse IP fixe permet de simplifier la gestion et la supervision du serveur. Les outils disponibles permettent de définir l’adresse, la passerelle ainsi que les DNS utilisés.

2.  Service DNS
Pour assurer la résolution de noms, nous avons déployé Bind9. Les zones directes et inverses ont été configurées pour traduire efficacement les noms de domaine locaux en adresses IP et inversement. Un système de forwarders permet de déléguer la résolution de noms inconnus à des serveurs DNS publics.

3.  Service LDAP
Un service d’annuaire LDAP est mis en place pour centraliser les identifiants et assurer une gestion unifiée des utilisateurs. L’outil permet d’organiser les comptes, groupes, machines et services au sein d’une hiérarchie logique (ou=Users, ou=Groups…).

4.  Serveurs Mail (Postfix & Dovecot)
La configuration mail repose sur Postfix pour l’envoi et la distribution, complété par Dovecot pour la réception et l’authentification IMAP/POP3. L’authentification LDAP permet une cohérence des identifiants au sein du réseau. Le chiffrement TLS est activé pour sécuriser la transmission.

5.  Partage & Domaine (Samba)
Le partage de fichiers et la gestion de domaine sous Samba ont été activés afin de proposer notamment des répertoires partagés aux utilisateurs, tout en conciliant la base d’authentification centralisée hébergée sur LDAP.

6.  Serveur Web
Un serveur Apache2 fournit un accès web, éventuellement protégé par une authentification LDAP. Cela permet un portail unique où les utilisateurs du réseau peuvent s’identifier et accéder à des ressources internes.

7.  Sécurité Générale
Des règles de pare-feu via iptables limitent l’accès au réseau. Par défaut, seuls les ports indispensables (53, 389, 80, 25, etc.) sont autorisés, et les communications SSH sont restreintes à des adresses approuvées. L’antivirus ClamAV assure la protection contre les malwares, notamment en scannant les fichiers reçus par mail ou présents dans les répertoires partagés.

## Table des Matières
- [IP Statique](#ip-statique)
- [DNS](#dns)
- [LDAP](#ldap)
- [Postfix](#postfix)
- [Dovecot](#dovecot)
- [Samba](#samba)
- [Serveur Web](#serveur-web)
- [Sécurité](#sécurité)

## IP Statique

### Outils requis : "network-manager"
```bash
apt install network-manager
```

### Afficher les Connexions et Modifier
Show connections and modify the one that you are connected to (or just 'modify Wired connection 1' if you are connected by a cable)
```bash
nmcli conn show
nmcli conn mod 'Wired connection 1' ipv4.method manual ipv4.addresses "10.2.200.200/16" ipv4.gateway "10.2.0.1" ipv4.dns "8.8.8.8,8.8.4.4" 
nmcli conn up 'Wired connection 1'
```

### Vérifier l’IP et Tester la Connectivité
```bash
ip a
ping google.com
```

## DNS

### Outils Requis
```bash
apt install bind9
apt install dnsutils  # recommended
```

### Configurer named.conf.options
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

### Créer des Fichiers de Zone
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
@       IN      A       10.2.200.200
@       IN      AAAA    ::1
```

### Créer un Fichier de Zone Inverse
```bash
cp db.127 db.10
nano db.10
```

Modify it to:
```
;
; BIND reverse data file for local 10.2.XXX.XXX interface
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
200.200     IN      PTR     groupe1.master2.fsa.ma.
```

### Modifier named.conf.local
Add:
```
zone "groupe1.master2.fsa.ma" {
        type master;
        file "/etc/bind/db.groupe1.master2.fsa.ma.zone";
};

zone "2.10.in-addr.arpa" {
        type master;
        file "/etc/bind/db.10";
};
```

### Redémarrer et Configurer le Réseau
```bash
systemctl restart bind9
nmcli conn mod 'Wired connection 1' ipv4.dns "127.0.0.1"
nmcli conn up 'Wired connection 1'
```

### Tester la Configuration
```bash
ping groupe1.master2.fsa.ma
nslookup groupe1.master2.fsa.ma localhost
dig -x 10.2.200.200
ping google.com
```

## LDAP

### Outils Requis
```bash
apt install slapd ldap-utils
```
When prompted to enter a password, just press enter.

### Reconfigurer LDAP
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

### Configurer LDAP
Edit `/etc/ldap/ldap.conf`:
```bash
nano /etc/ldap/ldap.conf
```

Uncomment and edit BASE and URI:
```
BASE    dc=groupe1,dc=master2,dc=fsa,dc=ma
URI     ldap://ldap.groupe1.master2.fsa.ma
```

### Ajouter des Entrées DNS
Add to `/etc/bind/db.groupe1.master2.fsa.ma.zone`:
```
ldap    IN      A       10.2.200.200
```

Add to `/etc/bind/db.10`:
```
200.200     IN      PTR     ldap.groupe1.master2.fsa.ma.
```

### Redémarrer et Tester
```bash
systemctl restart bind9
ping ldap.groupe1.master2.fsa.ma
ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config cn
ldapsearch -x -H ldap://ldap.groupe1.master2.fsa.ma -b dc=groupe1,dc=master2,dc=fsa,dc=ma
```

### Structure de Base
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

### Créer le Fichier de Base LDIF
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

### Ajouter des Utilisateurs
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

### Ajouter des Groupes
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

### Ajouter des Services
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

### Ajouter les Données à la Base
```bash
ldapadd -x -D "cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -f add_users.ldif
ldapadd -x -D "cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -f add_groups.ldif
ldapadd -x -D "cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -f add_services.ldif
```

### Installer et Configurer ldapscripts
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

### Créer un Modèle d’Utilisateur
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

### Configurer la Liste de Contrôle d’Accès
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

### Tester les Restrictions Utilisateur
To verify that users can see their own data:
```bash
ldapsearch -x -D "uid=john_doe,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -b "uid=john_doe,ou=Users,dc=groupe1,dc
```

To verify that users cannot see other users' data (should display '32 No such Object'):
```bash
ldapsearch -x -D "uid=john_doe,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -b "uid=jane_doe,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma"
```

### Vérifier l’Accès Utilisateur Anonyme
To verify anonymous users can only see general data but not sensitive data:
```bash
ldapmodify -x -D "uid=admin,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W
```

### Sécuriser LDAP avec TLS

#### Créer des Certificats TLS
```bash
openssl genrsa -out CA.key 8192 && chmod 400 CA.key
openssl req -new -x509 -nodes -key CA.key -days 3650 -out CA.pem
openssl genrsa -out device.key 4096 && chmod 400 device.key
openssl req -new -key device.key -out device.csr
openssl x509 -req -in device.csr -CA CA.pem -CAkey CA.key -CAcreateserial -out device.crt -days 365
```

#### Sauvegarder les Certificats
```bash
mkdir /etc/ldap/tls/
cp CA.pem device.key device.crt /etc/ldap/tls/
chown -R openldap:openldap /etc/ldap/tls/
chmod 101 /etc/ldap/tls/
chmod 400 /etc/ldap/tls/*
chmod 404 /etc/ldap/tls/CA.pem
```

#### Activer la Configuration TLS
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

### Tester la Connexion TLS
```bash
ldapsearch -LLL -x -H ldap://ldap.groupe1.master2.fsa.ma -b dc=groupe1,dc=master2,dc=fsa,dc=ma -ZZ
```

## Postfix

### Installation
```bash
apt install postfix postfix-ldap
apt install mailutils
```

When prompted:
- Select "Internet site" as initial type of configuration
- Enter mail.groupe1.master2.fsa.ma as your mail host

### Configuration DNS
Add to `/etc/bind/db.groupe1/master2.fsa.ma.zone`:
```
mail    IN    A    10.2.200.200
@       IN    MX   10 mail.groupe1.master2.fsa.ma.
```

Add to `/etc/bind/db.10`:
```
200.200    IN    PTR    mail.groupe1.master2.fsa.ma.
```

Restart BIND:
```bash
systemctl restart bind9
```

### Configuration du Schéma LDAP
```bash
wget -O postfix.ldif https://raw.githubusercontent.com/68b32/postfix-ldap-schema/master/postfix.ldif
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f postfix.ldif
```

### Tester l’Installation d’Utilisateur
```bash
ldapdeleteuser john_doe
ldapadduser john_doe Students
ldappasswd john_doe
```

### Fichiers de Configuration Postfix LDAP

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

### Déployer les Fichiers de Configuration
```bash
mkdir /etc/postfix/ldap
cp virtual* smtpd_sender_login_maps /etc/postfix/ldap/
```

### Tester la Configuration
```bash
postmap -q john_doe@groupe1.master2.fsa.ma ldap:/etc/postfix/ldap/virtual_mailbox_maps
postmap -q john_doe@groupe1.master2.fsa.ma ldap:/etc/postfix/ldap/virtual_uid_maps
postmap -q john_doe@groupe1.master2.fsa.ma ldap:/etc/postfix/ldap/virtual_alias_maps
```

### Sécuriser les Fichiers de Configuration
```bash
chown postfix:postfix /etc/postfix/ldap/*
chmod 400 /etc/postfix/ldap/*
```

### Configuration TLS
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

### Configuration Principale de Postfix
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

### Créer le Répertoire Mail
```bash
mkdir -p /home/mail
chmod o+w /home/mail
```

### Recharger Postfix
```bash
systemctl reload postfix
```

## Dovecot

### Installation
```bash
apt install dovecot-core dovecot-imapd dovecot-ldap
```

### Configuration de Base
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

### Configuration d’Authentification
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

### Configuration SSL/TLS
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

### Configurer la Socket d’Authentification
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

### Configurer SASL pour Postfix
Edit `/etc/postfix/main.cf`:
```
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
broken_sasl_auth_clients = yes
```



### Tester l’Authentification:
```
openssl s_client -starttls smtp -connect mail.groupe1.master2.fsa.ma:25
```
## Samba

### Installer le Logiciel

There are two packages needed when integrating Samba with LDAP: `samba` and `smbldap-tools`.

Strictly speaking, the `smbldap-tools` package isn’t needed, but unless you have some other way to manage the various Samba entities (users, groups, computers) in an LDAP context, install it:

```bash
sudo apt install samba smbldap-tools
```

### Configurer LDAP pour Samba

#### Tasks

1. Import a schema
2. Index some entries
3. Add objects

#### Samba Schema

In order for OpenLDAP to be used as a backend for Samba, the DIT needs attributes that can describe Samba data. Introduce a Samba LDAP schema:

```bash
sudo ldapadd -Q -Y EXTERNAL -H ldapi:/// -f /usr/share/doc/samba/examples/LDAP/samba.ldif
```

To query and view this new schema:

```bash
sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=schema,cn=config 'cn=*samba*'
```

#### Samba Indices

Set up some indices to improve performance during filtered searches on the DIT. Create the file `samba_indices.ldif` with the following content:

```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcDbIndex
olcDbIndex: objectClass eq
olcDbIndex: uidNumber,gidNumber eq
olcDbIndex: loginShell eq
olcDbIndex: uid,cn eq,sub
olcDbIndex: memberUid eq,sub
olcDbIndex: member,uniqueMember eq
olcDbIndex: sambaSID eq
olcDbIndex: sambaPrimaryGroupSID eq
olcDbIndex: sambaGroupType eq
olcDbIndex: sambaSIDList eq
olcDbIndex: sambaDomainName eq
olcDbIndex: default sub,eq
```

Load the new indices using `ldapmodify`:

```bash
sudo ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f samba_indices.ldif
```

Verify the new indices:

```bash
sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config olcDatabase={1}mdb olcDbIndex
```

#### Adding Samba LDAP Objects

Configure the `smbldap-tools` package to match your environment. First, decide on two settings in `/etc/samba/smb.conf`:

- **netbios name**: Server's name (default is derived from the hostname, truncated at 15 characters).
- **workgroup**: Workgroup name for the server or domain.

Generate the configuration:

```bash
sudo smbldap-config
```

Key settings:

- **workgroup name**: Must match `/etc/samba/smb.conf`.
- **ldap suffix**: Matches the LDAP suffix from LDAP server configuration.
- **other ldap suffixes**: For example, `ou=Users` for users and `ou=Computers` for machines.
- **ldap master bind dn and bind password**: Use Root DN credentials.

Run the population script:

```bash
sudo smbldap-populate -g 10000 -u 10000 -r 10000
```

To generate a LDIF file:

```bash
sudo smbldap-populate -e samba.ldif
```

Verify and rerun without the `-e` switch if everything is correct.

### Configuration de Samba

Edit `/etc/samba/smb.conf` to configure Samba to use LDAP. Comment out the default `passdb backend` parameter and add the following:

```ini
# passdb backend = tdbsam

workgroup = groupe1.master2.fsa.ma

# LDAP Settings

passdb backend = ldapsam:ldap://ldap.groupe1.master2.fsa.ma
ldap suffix = dc=groupe1,dc=master2,dc=fsa,dc=ma
ldap user suffix = ou=Users
ldap group suffix = ou=Groups
ldap machine suffix = ou=Computers
ldap idmap suffix = ou=Idmap
ldap admin dn = cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma
ldap ssl = start_tls
ldap passwd sync = yes
```

Inform Samba about the Root DN user’s password:

```bash
sudo smbpasswd -W
```

### Utiliser SSSD

Install and configure `sssd-ldap` to ensure LDAP users appear as Unix users:

```bash
sudo apt install sssd-ldap
```

Edit `/etc/sssd/sssd.conf`:

```ini
[sssd]
config_file_version = 2
domains = groupe1.master2.fsa.ma

[domain/groupe1.master2.fsa.ma]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://ldap.groupe1.master2.fsa.ma
cache_credentials = True
ldap_search_base = dc=groupe1,dc=master2,dc=fsa,dc=ma
```

Adjust permissions and start the service:

```bash
sudo chmod 0600 /etc/sssd/sssd.conf
sudo chown root:root /etc/sssd/sssd.conf
sudo systemctl start sssd
```

Restart Samba services:

```bash
sudo systemctl restart smbd.service nmbd.service
```

Test the setup:

```bash
getent group Replicators
```

### Gérer les Utilisateurs LDAP Existants

Add Samba attributes to existing LDAP users:

```bash
sudo smbpasswd -a username
```

### Gérer les Comptes avec smbldap-tools

- Add a new user:

  ```bash
  sudo smbldap-useradd -a -P -m username
  ```

- Remove a user:

  ```bash
  sudo smbldap-userdel username
  ```

- Add a group:

  ```bash
  sudo smbldap-groupadd -a groupname
  ```

- Add a user to a group:

  ```bash
  sudo smbldap-groupmod -m username groupname
  ```

- Remove a user from a group:

  ```bash
  sudo smbldap-groupmod -x username groupname
  ```

- Add a Samba machine account:

  ```bash
  sudo smbldap-useradd -t 0 -w machinename
  ```

## Serveur Web

### Outils Requis
```bash
apt install apache2
```

### Activer l’Authentification LDAP
```bash
a2enmod authnz_ldap
systemctl restart apache2
```

### Configurer l’Hôte Virtuel
Create `/etc/apache2/sites-available/groupe1.master2.fsa.ma.conf`:
```
<VirtualHost *:80>
    ServerName groupe1.master2.fsa.ma
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    # LDAP Authentication Configuration
    <Location />
        AuthType Basic
        AuthName "Restricted Content"
        AuthBasicProvider ldap
        AuthLDAPURL ldap://ldap.groupe1.master2.fsa.ma/dc=groupe1,dc=master2,dc=fsa,dc=ma?uid
        AuthLDAPBindDN "cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma"
        AuthLDAPBindPassword "toor"
        Require valid-user
    </Location>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

### Activer le Site et Redémarrer
```bash
a2ensite groupe1.master2.fsa.ma.conf
systemctl restart apache2
```

### Créer une Page d’Accueil Personnalisée
Create `/var/www/html/index.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Network Setup for Room 87</title>
    <style>
    header {
        background: #333;
        color: #fff;
        text-align: center;
        padding: 20px 0;
        box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
    }

    h1 {
        margin: 0;
        font-size: 3em;
        font-weight: bold;
    }

    section {
        max-width: 800px;
        margin: 40px auto;
        padding: 20px;
        background: #fff;
        border-radius: 10px;
        box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    }

    section p {
        font-size: 1.2em;
        line-height: 1.6;
        margin-bottom: 20px;
    }

    ul {
        list-style: none;
        padding: 0;
    }

    ul li {
        font-size: 1.1em;
        margin: 10px 0;
        padding: 10px;
        background: #f4f4f9;
        border-left: 5px solid #84fab0;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    }

    footer {
        text-align: center;
        padding: 10px;
        background: #333;
        color: #fff;
        font-size: 0.9em;
        position: fixed;
        bottom: 0;
        width: 100%;
    }
    </style>
</head>
<body>
    <header>
        <h1>Welcome to Room 87 Network</h1>
    </header>
    <section>
        <p>This project involves the configuration and management of essential network services:</p>
        <ul style="list-style-type: none; padding: 0;">
            <li>- DNS setup with <strong>Bind9</strong></li>
            <li>- Centralized authentication using <strong>LDAP</strong></li>
            <li>- Mail services powered by <strong>Postfix</strong> and <strong>Dovecot</strong></li>
            <li>- File sharing and domain control via <strong>Samba</strong></li>
            <li>- A Webserver with authentication to LDAP via <strong>Apache2</strong></li>
            <li>- Secure and efficient IP filtering with <strong>iptables</strong></li>
        </ul>
        <p>This environment ensures secure, efficient, and centralized management for Room 87.</p>
    </section>
</body>
</html>
```

Access the site at http://groupe1.master2.fsa.ma using LDAP credentials.

## Sécurité

### Outils Requis
```bash
apt install iptables iptables-persistent
```

### Configurer IP Tables

#### Contrôle d’Accès Réseau
Restrict access to local network:
```bash
iptables -A INPUT -s 10.2.0.0/16 -j ACCEPT
iptables -A INPUT -j DROP
```

#### Règles Spécifiques aux Services
Allow essential services:
```bash
# DNS
iptables -A INPUT -p tcp --dport 53 -j ACCEPT

# LDAP
iptables -A INPUT -p tcp --dport 389 -j ACCEPT
iptables -A INPUT -p tcp --dport 636 -j ACCEPT

# Apache
iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Postfix
iptables -A INPUT -p tcp --dport 25 -j ACCEPT
iptables -A INPUT -p tcp --dport 465 -j ACCEPT
iptables -A INPUT -p tcp --dport 587 -j ACCEPT

# Dovecot
iptables -A INPUT -p tcp --dport 143 -j ACCEPT
iptables -A INPUT -p tcp --dport 993 -j ACCEPT
iptables -A INPUT -p tcp --dport 110 -j ACCEPT
iptables -A INPUT -p tcp --dport 995 -j ACCEPT

# Samba
iptables -A INPUT -p tcp --dport 445 -j ACCEPT
iptables -A INPUT -p tcp --dport 139 -j ACCEPT
iptables -A INPUT -p tcp --dport 138 -j ACCEPT
iptables -A INPUT -p tcp --dport 137 -j ACCEPT
```

#### Contrôle d’Accès SSH
Restrict SSH access:
```bash
iptables -A INPUT -p tcp --dport 22 -s 10.2.100.100 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```

#### Gestion de l’État des Connexions
```bash
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### Configuration Antivirus

#### Installer ClamAV
```bash
apt install clamav clamav-daemon
```

#### Activer l’Analyse en Temps Réel
```bash
systemctl start clamav-daemon
systemctl enable clamav-daemon
```

#### Programmer des Analyses Quotidiennes
Create `/etc/cron.daily/clamav-scan`:
```bash
#!/bin/bash
clamscan -r /home/mail --log=/var/log/clamav/daily-scan.log --move=/quarantine
clamscan -r /srv/samba/shared --log=/var/log/clamav/daily-scan.log --move=/quarantine
```

Set permissions:
```bash
chmod +x /etc/cron.daily/clamav-scan
```

#### Intégration au Serveur Mail
Install ClamAV Milter:
```bash
apt install clamav-milter
```

Edit `/etc/postfix/main.cf`:
```
content_filter = scan:127.0.0.1:10026
```

Restart Postfix:
```bash
systemctl restart postfix
```
