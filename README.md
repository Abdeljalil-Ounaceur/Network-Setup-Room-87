# Configuration Réseau Salle 87
_Utilisation de XUbuntu_24 LTS_  
_Sous-réseau 10.2.0.0/16_  
_Utilisé l’interface 'Wired connection 1'_  

# Objectif
Ce document de synthèse présente la configuration complète de plusieurs services réseau pour la salle 87. Dans l’objectif d’offrir un réseau fiable et sécurisé, différentes briques logicielles ont été mises en place pour gérer l’adressage IP, la résolution de noms, l’authentification centralisée, la messagerie, le partage de ressources et la sécurité globale du système.

1.  **Configuration IP Statique**\
Le choix d’une adresse IP fixe permet de simplifier la gestion et la supervision du serveur. Les outils disponibles permettent de définir l’adresse, la passerelle ainsi que les DNS utilisés.

2.  **Service DNS**\
Pour assurer la résolution de noms, nous avons déployé Bind9. Les zones directes et inverses ont été configurées pour traduire efficacement les noms de domaine locaux en adresses IP et inversement. Un système de forwarders permet de déléguer la résolution de noms inconnus à des serveurs DNS publics.

3.  **Service LDAP**\
Un service d’annuaire LDAP est mis en place pour centraliser les identifiants et assurer une gestion unifiée des utilisateurs. L’outil permet d’organiser les comptes, groupes, machines et services au sein d’une hiérarchie logique (ou=Users, ou=Groups…).

4.  **Serveurs Mail (Postfix & Dovecot)**\
La configuration mail repose sur Postfix pour l’envoi et la distribution, complété par Dovecot pour la réception et l’authentification IMAP/POP3. L’authentification LDAP permet une cohérence des identifiants au sein du réseau. Le chiffrement TLS est activé pour sécuriser la transmission.

5.  **Partage & Domaine (Samba)**\
Le partage de fichiers et la gestion de domaine sous Samba ont été activés afin de proposer notamment des répertoires partagés aux utilisateurs, tout en conciliant la base d’authentification centralisée hébergée sur LDAP.

6.  **Serveur Web**\
Un serveur Apache2 fournit un accès web, éventuellement protégé par une authentification LDAP. Cela permet un portail unique où les utilisateurs du réseau peuvent s’identifier et accéder à des ressources internes.

7.  **Sécurité Générale**\
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
- [Conclusion](#conclusion)

## IP Statique

### Outils requis : "network-manager"
```bash
apt install network-manager
```

### Afficher les Connexions et Modifier
Afficher les connexions et modifier celle à laquelle vous êtes connecté (ou simplement 'modify Wired connection 1' si vous êtes connecté par câble).
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
apt install dnsutils  # recommandé
```

### Configurer named.conf.options
Éditez `named.conf.options` :
```bash
nano named.conf.options
```

Décommentez les forwarders et ajoutez :
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

Éditez le fichier de zone :
```bash
nano db.groupe1.master2.fsa.ma.zone
```

Remplacez toute occurrence de 'localhost' par 'groupe1.master2.fsa.ma' et remplacez 127.0.0.1 par votre adresse IP statique :
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

Modifiez le fichier comme suit :
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
Ajoutez :
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
Lorsque vous êtes invité à saisir un mot de passe, appuyez simplement sur Entrée.

### Reconfigurer LDAP
```bash
dpkg-reconfigure slapd
```

Entrez dans l'ordre:
- Non
- groupe1.master2.fsa.ma
- Groupe1 Master2 FSA
- some_password
- some_password (à nouveau)
- oui
- oui

### Configurer LDAP
Modifier `/etc/ldap/ldap.conf`:
```bash
nano /etc/ldap/ldap.conf
```

Décommenter et modifier BASE et URI:
```
BASE    dc=groupe1,dc=master2,dc=fsa,dc=ma
URI     ldap://ldap.groupe1.master2.fsa.ma
```

### Ajouter des Entrées DNS
Ajouter à `/etc/bind/db.groupe1.master2.fsa.ma.zone`:
```
ldap    IN      A       10.2.200.200
```

Ajouter à `/etc/bind/db.10`:
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
Créer `base_tree.ldif`:
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

Ajouter à la base de données:
```bash
ldapadd -x -D "cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -f base_tree.ldif
```

### Ajouter des Utilisateurs
Créer `add_users.ldif`:
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
Créer `add_groups.ldif`:
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
Créer `add_services.ldif`:
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
Ajouter à la base de données:
```bash
ldapadd -x -D "cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -f add_users.ldif
ldapadd -x -D "cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -f add_groups.ldif
ldapadd -x -D "cn=admin,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -f add_services.ldif
```

### Installer et Configurer ldapscripts
```bash
apt install ldapscripts
```

Modifier `/etc/ldapscripts/ldapscripts.conf`:
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

Enregistrer le mot de passe:
```bash
echo -n 'password' | sudo tee /etc/ldapscripts/ldapscripts.passwd
chmod 400 /etc/ldapscripts/ldapscripts.passwd
```

### Créer un Modèle d’Utilisateur
Modifier `/etc/ldapscripts/ldapscripts.conf`:
```
UTEMPLATE="/etc/ldapscripts/ldapadduser.template"
```

Créer `/etc/ldapscripts/ldapadduser.template`:
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
Créer `ACL.ldif`:
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

Appliquer l’ACL:
```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f ACL.ldif
```

### Tester les Restrictions Utilisateur
Pour vérifier que les utilisateurs peuvent voir leurs propres données:
```bash
ldapsearch -x -D "uid=john_doe,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -b "uid=john_doe,ou=Users,dc=groupe1,dc
```

Pour vérifier que les utilisateurs ne peuvent pas voir les données des autres utilisateurs (devrait afficher '32 No such Object'):
```bash
ldapsearch -x -D "uid=john_doe,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma" -W -b "uid=jane_doe,ou=Users,dc=groupe1,dc=master2,dc=fsa,dc=ma"
```

### Vérifier l’Accès Utilisateur Anonyme
Pour vérifier que les utilisateurs anonymes ne peuvent voir que les données générales et non les données sensibles:
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
Créer `TLS_enable.ldif`:
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

Activer TLS:
```bash
ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f TLS_enable.ldif
```

Modifier `/etc/ldap/ldap.conf`:
```
TLS_CACERT /etc/ldap/tls/CA.pem
```

### Tester la Connexion TLS
Pour tester la connexion TLS:
```bash
ldapsearch -LLL -x -H ldap://ldap.groupe1.master2.fsa.ma -b dc=groupe1,dc=master2,dc=fsa,dc=ma -ZZ
```

## Postfix

### Installation
```bash
apt install postfix postfix-ldap
apt install mailutils
```

Lorsque vous y êtes invité :
- Sélectionnez "Internet site" comme type initial de configuration
- Entrez mail.groupe1.master2.fsa.ma comme votre hôte de messagerie

### Configuration DNS
Ajouter à `/etc/bind/db.groupe1/master2.fsa.ma.zone` :
```
mail    IN    A    10.2.200.200
@       IN    MX   10 mail.groupe1.master2.fsa.ma.
```

Ajouter à `/etc/bind/db.10` :
```
200.200    IN    PTR    mail.groupe1.master2.fsa.ma.
```

Redémarrer BIND :
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

Créer le fichier `virtual_alias_domains` :
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

Créer le fichier `virtual_alias_maps` :
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

Créer le fichier `virtual_mailbox_maps` :
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

Créer le fichier `virtual_uid_maps` :
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

Créer le fichier `smtpd_sender_login_maps` :
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

Créer le fichier `/etc/systemd/system/var-spool-postfix-etc-ldap-tls-CA.pem.mount` :
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

Activer le montage TLS :
```bash
systemctl daemon-reload
systemctl start var-spool-postfix-etc-ldap-tls-CA.pem.mount
mount | grep CA
systemctl enable var-spool-postfix-etc-ldap-tls-CA.pem.mount
```

### Configuration Principale de Postfix
Éditer `/etc/postfix/main.cf` :
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
Éditez `/etc/dovecot/conf.d/10-mail.conf`:
```
maildir:~/mailbox
```

Éditez `/etc/dovecot/conf.d/10-master.conf`:
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
Éditez `/etc/dovecot/conf.d/10-auth.conf`:
```
#!include auth-system.conf.ext
!include auth-ldap.conf.ext
```

Éditez `/etc/dovecot/conf.d/auth-ldap.conf.ext`:
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

Éditez `/etc/dovecot/dovecot-ldap.conf.ext`:
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
Éditez `/etc/dovecot/conf.d/10-auth.conf`:
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
Éditez `/etc/dovecot/conf.d/10-master.conf`:
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
Éditez `/etc/postfix/main.cf`:
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

Il y a deux paquets nécessaires lors de l'intégration de Samba avec LDAP : `samba` et `smbldap-tools`.

Strictement parlant, le paquet `smbldap-tools` n'est pas indispensable, mais à moins que vous n'ayez un autre moyen de gérer les différentes entités Samba (utilisateurs, groupes, ordinateurs) dans un contexte LDAP, installez-le :

```bash
sudo apt install samba smbldap-tools
```

### Configurer LDAP pour Samba

#### Tâches

1. Importer un schéma
2. Indexer certaines entrées
3. Ajouter des objets

#### Schéma Samba

Afin qu'OpenLDAP puisse être utilisé comme backend pour Samba, le DIT a besoin d'attributs pouvant décrire les données Samba. Introduisez un schéma LDAP Samba :

```bash
sudo ldapadd -Q -Y EXTERNAL -H ldapi:/// -f /usr/share/doc/samba/examples/LDAP/samba.ldif
```

Pour interroger et visualiser ce nouveau schéma :

```bash
sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=schema,cn=config 'cn=*samba*'
```

#### Indices Samba

Configurez certains indices pour améliorer les performances lors des recherches filtrées sur le DIT. Créez le fichier `samba_indices.ldif` avec le contenu suivant :

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

Chargez les nouveaux index en utilisant `ldapmodify` :

```bash
sudo ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f samba_indices.ldif
```

Vérifiez les nouveaux index :

```bash
sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config olcDatabase={1}mdb olcDbIndex
```

#### Ajout d'objets LDAP Samba

Configurez le paquet `smbldap-tools` pour correspondre à votre environnement. Tout d'abord, décidez de deux paramètres dans `/etc/samba/smb.conf` :

- **netbios name**: Nom du serveur (par défaut dérivé du nom d'hôte, tronqué à 15 caractères).
- **workgroup**: Nom du groupe de travail pour le serveur ou le domaine.

Générez la configuration :

```bash
sudo smbldap-config
```

Paramètres clés :

- **workgroup name**: Doit correspondre à `/etc/samba/smb.conf`.
- **ldap suffix**: Correspond au suffixe LDAP de la configuration du serveur LDAP.
- **other ldap suffixes**: Par exemple, `ou=Users` pour les utilisateurs et `ou=Computers` pour les machines.
- **ldap master bind dn and bind password**: Utilisez les identifiants du Root DN.

Exécutez le script de population :

```bash
sudo smbldap-populate -g 10000 -u 10000 -r 10000
```

Pour générer un fichier LDIF :

```bash
sudo smbldap-populate -e samba.ldif
```

Vérifiez et relancez sans l’option `-e` si tout est correct.

### Configuration de Samba

Modifiez `/etc/samba/smb.conf` pour configurer Samba à utiliser LDAP. Commentez le paramètre `passdb backend` par défaut et ajoutez ce qui suit :

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

Informez Samba du mot de passe de l'utilisateur Root DN :

```bash
sudo smbpasswd -W
```

### Utiliser SSSD

Installez et configurez `sssd-ldap` pour vous assurer que les utilisateurs LDAP apparaissent comme des utilisateurs Unix :

```bash
sudo apt install sssd-ldap
```

Modifiez `/etc/sssd/sssd.conf` :

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

Ajustez les permissions et démarrez le service :

```bash
sudo chmod 0600 /etc/sssd/sssd.conf
sudo chown root:root /etc/sssd/sssd.conf
sudo systemctl start sssd
```

Redémarrez les services Samba :

```bash
sudo systemctl restart smbd.service nmbd.service
```

Testez la configuration :

```bash
getent group Replicators
```

### Gérer les Utilisateurs LDAP Existants

Ajoutez des attributs Samba aux utilisateurs LDAP existants :

```bash
sudo smbpasswd -a username
```

### Gérer les Comptes avec smbldap-tools

- Ajouter un nouvel utilisateur :

  ```bash
  sudo smbldap-useradd -a -P -m username
  ```

- Supprimer un utilisateur :

  ```bash
  sudo smbldap-userdel username
  ```

- Ajouter un groupe :

  ```bash
  sudo smbldap-groupadd -a groupname
  ```

- Ajouter un utilisateur à un groupe :

  ```bash
  sudo smbldap-groupmod -m username groupname
  ```

- Retirer un utilisateur d'un groupe :

  ```bash
  sudo smbldap-groupmod -x username groupname
  ```

- Ajouter un compte machine Samba :

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
Créer `/etc/apache2/sites-available/groupe1.master2.fsa.ma.conf`:
```
<VirtualHost *:80>
    ServerName groupe1.master2.fsa.ma
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    # Configuration de l'authentification LDAP
    <Location />
        AuthType Basic
        AuthName "Contenu Restreint"
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
Créer `/var/www/html/index.html`:
```html
<!DOCTYPE html>
<html lang="fr">
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
        <h1>Bienvenue sur le Réseau de la Salle 87</h1>
    </header>
    <section>
        <p>Ce projet consiste en la configuration et la gestion de services réseau essentiels :</p>
        <ul style="list-style-type: none; padding: 0;">
            <li>- Configuration DNS avec <strong>Bind9</strong></li>
            <li>- Authentification centralisée avec <strong>LDAP</strong></li>
            <li>- Services de courrier électronique gérés par <strong>Postfix</strong> et <strong>Dovecot</strong></li>
            <li>- Partage de fichiers et contrôle de domaine via <strong>Samba</strong></li>
            <li>- Un serveur web avec authentification LDAP via <strong>Apache2</strong></li>
            <li>- Un filtrage IP sécurisé et efficace avec <strong>iptables</strong></li>
        </ul>
        <p>Cet environnement garantit une gestion sécurisée, efficace et centralisée pour la Salle 87.</p>
    </section>
</body>
</html>
```

Accédez au site sur http://groupe1.master2.fsa.ma en utilisant les identifiants LDAP.

## Sécurité

### Outils Requis
```bash
apt install iptables iptables-persistent
```

### Configurer IP Tables

#### Contrôle d’Accès Réseau
Restreindre l'accès au réseau local:
```bash
iptables -A INPUT -s 10.2.0.0/16 -j ACCEPT
iptables -A INPUT -j DROP
```

#### Règles Spécifiques aux Services
Autoriser les services essentiels:
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
Restreindre l'accès SSH:
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
Créer `/etc/cron.daily/clamav-scan`:
```bash
#!/bin/bash
clamscan -r /home/mail --log=/var/log/clamav/daily-scan.log --move=/quarantine
clamscan -r /srv/samba/shared --log=/var/log/clamav/daily-scan.log --move=/quarantine
```

Définir les permissions:
```bash
chmod +x /etc/cron.daily/clamav-scan
```

#### Intégration au Serveur Mail
Installer ClamAV Milter:
```bash
apt install clamav-milter
```

Modifier `/etc/postfix/main.cf`:
```
content_filter = scan:127.0.0.1:10026
```

Redémarrer Postfix:
```bash
systemctl restart postfix
```

## Conclusion

Ce document présente la configuration actuelle des services réseau pour la salle 87. Ce rapport sera amélioré à l'avenir pour inclure des tests de fonctionnalité et ajouter des sections concernant des mesures de sécurité supplémentaires pour le serveur.
