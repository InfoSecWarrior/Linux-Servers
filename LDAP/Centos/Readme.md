# LDAP in CentOS 9

This document outlines the steps for installing and configuring an OpenLDAP server on CentOS 9 , as well as configuring a client to use LDAP authentication. 

--------------------------------------------------------------------------------
## 1. Server Setup
--------------------------------------------------------------------------------

### 1.1 Set Hostname
• This command sets the system’s hostname to "centos.armour.local" for DNS resolution and server identification.

```
hostnamectl set-hostname centos.armour.local
```

--------------------------------------------------------------------------------
### 1.2 Install & Configure DNS (BIND)
--------------------------------------------------------------------------------

#### 1.2.1 Install DNS Packages
• Installs the BIND DNS server and related utilities for name resolution.

```
dnf install -y bind bind-utils
```

#### 1.2.2 Edit main named configuration
• Configures the named service’s global options such as listen address, allowed queries, and DNS forwarders.

```
vim /etc/named.conf
```
```conf
options {
    listen-on port 53 { 127.0.0.1; 192.168.1.128; };
    allow-query { localhost; 192.168.1.0/24; };
    forwarders { 8.8.8.8; 8.8.4.4; };
};
```

#### 1.2.3 Create local includes for DNS zones
• Defines the DNS zones for "armour.local" and the reverse lookup zone.

```
vim /etc/named.conf.local
```
```conf
zone "armour.local" {
    type master;
    file "/var/named/armour.local.zone";
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/var/named/1.168.192.rev";
};
```

#### 1.2.4 Configure Forward Zone
• Specifies forward lookup data for the domain "armour.local".

```
vim /var/named/armour.local.zone
```
```conf
$TTL 86400
@   IN  SOA centos.armour.local. admin.armour.local. (
        2024010101  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400       ; Minimum TTL
)
    IN  NS  centos.armour.local.
    IN  A   192.168.1.128

centos  IN  A   192.168.1.128
ldap    IN  CNAME   centos
```

#### 1.2.5 Configure Reverse Zone
• Specifies reverse lookup data for "192.168.1.x" addresses.

```
vim /var/named/1.168.192.rev
```
```conf
$TTL 86400
@   IN  SOA centos.armour.local. admin.armour.local. (
        2024010101  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400       ; Minimum TTL
)
    IN  NS  centos.armour.local.

128  IN  PTR centos.armour.local.
```

#### 1.2.6 Enable & Allow DNS Service
• Starts named and configures the firewall to permit DNS traffic.

```
systemctl enable named --now
```

```
firewall-cmd --add-service=dns --permanent
```

```
systemctl restart named firewalld
```

#### 1.2.7 Verify DNS
• Tests the DNS resolution for the hostname.

```
dig centos.armour.local
```

--------------------------------------------------------------------------------
## 2. Configure Required Repositories
--------------------------------------------------------------------------------

### 2.1 Enable EPEL Repository
• Installs the EPEL repo, providing additional packages for Enterprise Linux.

```
dnf install -y epel-release
```

### 2.2 Enable CRB (CodeReady Builder) Repository
• Sets the "crb" repo to an enabled state for required dependencies.

```
dnf config-manager --set-enabled crb
```

### 2.3 Update Repository Cache
• Refreshes the local package metadata for the newly enabled repositories.

```
dnf makecache
```

--------------------------------------------------------------------------------
## 3. Install OpenLDAP Packages
--------------------------------------------------------------------------------

### 3.1 Install OpenLDAP Server & Clients
• Installs the main OpenLDAP server daemon (slapd) and command-line LDAP clients.

```
dnf install -y openldap-servers openldap-clients
```

### 3.2 Install Additional LDAP Utilities
• Provides development libraries and additional SSSD tools for LDAP management.

```
dnf install -y openldap-devel sssd-tools
```

### 3.3 Verify Installation
• Confirms that OpenLDAP packages are correctly installed.

```
rpm -qa | grep openldap
```

### 3.4 Enable & Start slapd
• Starts the LDAP server and enables it to run on system boot.

```
systemctl enable slapd --now
```

```
systemctl status slapd
```

#### Troubleshooting (Service Fails)  
• Checks the system journal for errors if slapd fails to start.

```
journalctl -xeu slapd
```

--------------------------------------------------------------------------------
## 4. Firewall Configuration
--------------------------------------------------------------------------------

### 4.1 Allow LDAP Services
• Opens standard LDAP (389) and LDAPS (636) ports in the firewall.

```
firewall-cmd --add-service=ldap --permanent
```

```
firewall-cmd --add-service=ldaps --permanent
```

```
firewalld
```

```
ss -nltup | grep slapd
```

--------------------------------------------------------------------------------
## 5. Test Basic LDAP Connectivity
--------------------------------------------------------------------------------

### 5.1 Anonymous LDAP Search
• Verifies that the LDAP server is responding to queries anonymously.

```
ldapsearch -x -H ldap://localhost -b "" -s base
```

--------------------------------------------------------------------------------
## 6. Making LDAP Tree
--------------------------------------------------------------------------------

### 6.1 Generate Root Password Hash
• Creates a hashed password for the LDAP root account.

```
slappasswd
```

--------------------------------------------------------------------------------
### 6.2 Configure the LDAP Database
--------------------------------------------------------------------------------

#### 6.2.1 Database Suffix & RootDN
• Modifies the default LDAP database settings (suffix & admin DN) and sets the LDAP admin password.

```
vim  /root/ldif/db-config.ldif
```
```ldif
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=armour,dc=local

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=armour,dc=local

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}YOUR_PASSWORD_HASH_HERE
```

```
ldapmodify -Y EXTERNAL -H ldapi:/// -f /root/ldif/db-config.ldif
```

--------------------------------------------------------------------------------
### 6.3 Load Required Schemas
--------------------------------------------------------------------------------

#### 6.3.1 Standard LDAP Schemas
• Adds cosine, nis, and inetorgperson schemas needed for user and group support.

```
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
```

```
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
```

```
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

--------------------------------------------------------------------------------
### 6.4 Set up Access Control Lists (ACLs)
--------------------------------------------------------------------------------

#### 6.4.1 ACL Configuration File
• Controls who can read, write, and authenticate against the LDAP directory.

```
vim /root/ldif/acl.ldif
```
```ldif
dn: olcDatabase={2}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword
  by self write
  by anonymous auth
  by dn="cn=admin,dc=armour,dc=local" write
  by * none
olcAccess: {1}to attrs=shadowLastChange
  by self write
  by * read
olcAccess: {2}to *
  by dn="cn=admin,dc=armour,dc=local" write
  by self read
  by * read
```

#### 6.4.2 Apply ACLs
• Loads the above-defined ACLs into the LDAP configuration.

```
ldapmodify -Y EXTERNAL -H ldapi:/// -f /root/ldif/acl.ldif
```

#### 6.4.3 Verify ACLs
• Retrieves current ACL settings from the LDAP configuration.

```
ldapsearch -Y EXTERNAL -H ldapi:/// -b "olcDatabase={2}mdb,cn=config" olcAccess
```

--------------------------------------------------------------------------------
### 6.5 Create Base Domain
--------------------------------------------------------------------------------

#### 6.5.1 Base LDAP Entry
• Establishes the top-level domain entry "dc=armour,dc=local."

```
vim /root/ldif/base.ldif
```
```ldif
dn: dc=armour,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: Armour
dc: armour
```

#### 6.5.2 Add Base Domain to LDAP
• Adds the new base domain into the LDAP database.

```
ldapadd -x -D "cn=admin,dc=armour,dc=local" -W -f /root/ldif/base.ldif
```

--------------------------------------------------------------------------------
### 6.6 Create Organizational Units
--------------------------------------------------------------------------------

#### 6.6.1 OU Configuration
• Adds organizational units for "it", "hr", "admin", and "groups" under the base domain.

```
vim /root/ldif/base_structure.ldif
```
```ldif
dn: ou=it,dc=armour,dc=local
objectClass: organizationalUnit
ou: it

dn: ou=hr,dc=armour,dc=local
objectClass: organizationalUnit
ou: hr

dn: ou=admin,dc=armour,dc=local
objectClass: organizationalUnit
ou: admin

dn: ou=groups,dc=armour,dc=local
objectClass: organizationalUnit
ou: groups
```

#### 6.6.2 Add OUs to LDAP
• Loads the OU entries into the LDAP directory.

```
ldapadd -x -D "cn=admin,dc=armour,dc=local" -W -f /root/ldif/base_structure.ldif
```

--------------------------------------------------------------------------------
### 6.7 Create Groups
--------------------------------------------------------------------------------

#### 6.7.1 Group Configuration
• Defines posixGroups under "ou=groups,dc=armour,dc=local."

```
vim /root/ldif/create_group.ldif
```
```ldif
dn: cn=it,ou=groups,dc=armour,dc=local
objectClass: posixGroup
cn: it
gidNumber: 2004

dn: cn=hr,ou=groups,dc=armour,dc=local
objectClass: posixGroup
cn: hr
gidNumber: 2005

dn: cn=admin,ou=groups,dc=armour,dc=local
objectClass: posixGroup
cn: admin
gidNumber: 2003
```

#### 6.7.2 Add Groups to LDAP
• Inserts the group definitions into the LDAP database.

```
ldapadd -x -D "cn=admin,dc=armour,dc=local" -W -f /root/ldif/create_group.ldif
```

--------------------------------------------------------------------------------
### 6.8 Create Users
--------------------------------------------------------------------------------

#### 6.8.1 Generate User Password Hash
• Provides a hashed password for new LDAP users.

```
slappasswd
```

#### 6.8.2 User Entries File
• Defines LDAP user objects for hr1, hr2, it1, it2, admin1, and admin2.

```
vim /root/ldif/add_users.ldif
```
```ldif
dn: uid=hr1,ou=hr,dc=armour,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
cn: HR One
sn: One
uid: hr1
uidNumber: 3001
gidNumber: 2005
homeDirectory: /home/hr1
loginShell: /bin/bash
userPassword: {SSHA}8ZFc/mjJqAzQ0Oeebegd7XX4aXfj6riI

dn: uid=hr2,ou=hr,dc=armour,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
cn: HR Two
sn: Two
uid: hr2
uidNumber: 3002
gidNumber: 2005
homeDirectory: /home/hr2
loginShell: /bin/bash
userPassword: {SSHA}8ZFc/mjJqAzQ0Oeebegd7XX4aXfj6riI

dn: uid=it1,ou=it,dc=armour,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
cn: IT One
sn: One
uid: it1
uidNumber: 3003
gidNumber: 2004
homeDirectory: /home/it1
loginShell: /bin/bash
userPassword: {SSHA}8ZFc/mjJqAzQ0Oeebegd7XX4aXfj6riI

dn: uid=it2,ou=it,dc=armour,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
cn: IT Two
sn: Two
uid: it2
uidNumber: 3004
gidNumber: 2004
homeDirectory: /home/it2
loginShell: /bin/bash
userPassword: {SSHA}8ZFc/mjJqAzQ0Oeebegd7XX4aXfj6riI

dn: uid=admin1,ou=admin,dc=armour,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
cn: Admin One
sn: One
uid: admin1
uidNumber: 3005
gidNumber: 2003
homeDirectory: /home/admin1
loginShell: /bin/bash
userPassword: {SSHA}8ZFc/mjJqAzQ0Oeebegd7XX4aXfj6riI

dn: uid=admin2,ou=admin,dc=armour,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
cn: Admin Two
sn: Two
uid: admin2
uidNumber: 3006
gidNumber: 2003
homeDirectory: /home/admin2
loginShell: /bin/bash
userPassword: {SSHA}8ZFc/mjJqAzQ0Oeebegd7XX4aXfj6riI
```

#### 6.8.3 Add Users to LDAP
• Loads the defined users into the LDAP directory.

```
ldapadd -x -D "cn=admin,dc=armour,dc=local" -W -f /root/ldif/add_users.ldif
```

--------------------------------------------------------------------------------
### 6.9 Test Search & Authentication
--------------------------------------------------------------------------------

#### 6.9.1 Test Search All Entries
• Searches the entire domain for any objectClass, verifying read access.

```
ldapsearch -x -D "cn=admin,dc=armour,dc=local" -W -b "dc=armour,dc=local" "(objectClass=*)"
```

#### 6.9.2 Test User Authentication
• Checks that a specific LDAP user (it1) can authenticate using the supplied credentials.

```
ldapwhoami -x -D "uid=it1,ou=it,dc=armour,dc=local" -W
```

--------------------------------------------------------------------------------
### 6.10 Link Users to Groups
--------------------------------------------------------------------------------

#### 6.10.1 Group Membership File
• Adds each user as a memberUid to the corresponding group entries.

```
vim /root/ldif/add_member_to_group.ldif
```
```ldif
dn: cn=it,ou=groups,dc=armour,dc=local
changetype: modify
add: memberUid
memberUid: it1
memberUid: it2

dn: cn=hr,ou=groups,dc=armour,dc=local
changetype: modify
add: memberUid
memberUid: hr1
memberUid: hr2

dn: cn=admin,ou=groups,dc=armour,dc=local
changetype: modify
add: memberUid
memberUid: admin1
memberUid: admin2
```

#### 6.10.2 Apply Group Membership
• Modifies existing groups in the LDAP directory to reference the new users.

```
ldapmodify -x -D "cn=admin,dc=armour,dc=local" -W -f /root/ldif/add_member_to_group.ldif
```

#### 6.10.3 Verify Group Membership
• Searches for all posixGroup objects to confirm the users were added.

```
ldapsearch -x -D "cn=admin,dc=armour,dc=local" -W -b "ou=groups,dc=armour,dc=local" "(objectClass=posixGroup)"
```

--------------------------------------------------------------------------------
### 6.11 Enable Logging (Optional)
--------------------------------------------------------------------------------

#### 6.11.1 Configure Logging
• Controls the LDAP server’s logging level (stats in this case).

```
vim /root/ldif/logging.ldif
```
```ldif
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: stats
```

#### 6.11.2 Apply Logging Settings
• Applies the logging configuration to the LDAP server.

```
ldapmodify -Y EXTERNAL -H ldapi:/// -f /root/ldif/logging.ldif
```

--------------------------------------------------------------------------------
### 6.12 Final Tests
--------------------------------------------------------------------------------

#### 6.12.1 Test Admin Authentication
• Verifies that the LDAP admin DN can bind successfully.

```
ldapwhoami -x -D "cn=admin,dc=armour,dc=local" -W
```

#### 6.12.2 Search All Entries
• Ensures the LDAP directory is accessible under the admin credentials.

```
ldapsearch -x -D "cn=admin,dc=armour,dc=local" -W -b "dc=armour,dc=local" "(objectClass=*)"
```

#### 6.12.3 Test User Authentication (it1)
• Checks user-level authentication once more.

```
ldapwhoami -x -D "uid=it1,ou=it,dc=armour,dc=local" -W
```

#### 6.12.4 View Logs
• Observes real-time slapd logs for any additional debugging.

```
journalctl -u slapd -f
```

--------------------------------------------------------------------------------
### 6.13 LDAP Tree Structure
--------------------------------------------------------------------------------

```
dc=armour,dc=local
├── ou=hr
│   ├── uid=hr1 (member of hr group)
│   └── uid=hr2 (member of hr group)
├── ou=it
│   ├── uid=it1 (member of it group)
│   └── uid=it2 (member of it group)
├── ou=admin
│   ├── uid=admin1 (member of admin group)
│   └── uid=admin2 (member of admin group)
└── ou=groups
    ├── cn=hr (memberUid: hr1, hr2)
    ├── cn=it (memberUid: it1, it2)
    └── cn=admin (memberUid: admin1, admin2)
```

--------------------------------------------------------------------------------
# Client Side Configuration
--------------------------------------------------------------------------------

### 1. Set Client Hostname
• Assigns the client system a hostname for integration with LDAP.

```
hostnamectl set-hostname c1.armour.local
```

### 2. Configure DNS to Use Your Server
• Instructs the client to use the LDAP server’s DNS for hostname resolution.

```
nmtui
```
*(Set DNS to 192.168.1.128)*

### 3. Verify DNS Resolution
• Ensures the client can resolve the LDAP server’s hostname.

```
nslookup centos.armour.local
```

--------------------------------------------------------------------------------
## 4. Install Required Packages
--------------------------------------------------------------------------------

• Installs the OpenLDAP client tools, SSSD services, and oddjob-mkhomedir for home directory creation.

```
dnf install openldap-clients sssd sssd-ldap oddjob-mkhomedir sssd-tools -y
```

--------------------------------------------------------------------------------
## 5. Configure SSSD
--------------------------------------------------------------------------------

### 5.1 SSSD Configuration File
• Specifies how SSSD connects to the LDAP server, including credentials, schema, and enumeration.

```
vim /etc/sssd/sssd.conf
```
```ini
[sssd]
config_file_version = 2
services = nss, pam, sudo
domains = armour.local

[domain/armour.local]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://centos.armour.local:389
ldap_search_base = dc=armour,dc=local
ldap_schema = rfc2307
ldap_tls_reqcert = never
ldap_auth_disable_tls_never_use_in_production = true
ldap_id_use_start_tls = false
cache_credentials = true
enumerate = true
ldap_default_bind_dn = cn=admin,dc=armour,dc=local
ldap_default_authtok = @rmour123

[nss]
homedir_substring = /home
filter_groups = root
filter_users = root

[pam]
offline_credentials_expiration = 60

[sudo]
```

#### Configuration Details
- **ldap_uri** - LDAP server location
- **ldap_search_base** - Base DN for searches
- **ldap_default_bind_dn** - Admin DN for LDAP queries
- **ldap_default_authtok** - Admin password
- **ldap_auth_disable_tls_never_use_in_production** - Disables TLS for authentication (remove when implementing LDAPS)


--------------------------------------------------------------------------------
### 5.2 Set Secure Permissions
--------------------------------------------------------------------------------

#### 5.2.1 Protect sssd.conf
• Restricts access to the SSSD configuration file to the root user only.

```
chmod 600 /etc/sssd/sssd.conf
```

```
chown root:root /etc/sssd/sssd.conf
```

--------------------------------------------------------------------------------
### 5.3 Enable & Start SSSD
--------------------------------------------------------------------------------

#### 5.3.1 SSSD Service
• Activates the SSSD service, which handles LDAP authentication for the client.

```
systemctl enable sssd --now
```

```
systemctl status sssd
```

#### Troubleshooting Commands (SSSD Fails)
• Checks for recent log messages related to SSSD errors.

```
journalctl -u sssd -n 50
```

--------------------------------------------------------------------------------
## 6. Configure PAM to Use SSSD
--------------------------------------------------------------------------------

• Integrates SSSD with the system authentication, including automatic home directory creation.

```
authselect select sssd with-mkhomedir --force
```

--------------------------------------------------------------------------------
## 7. Start oddjobd Service
--------------------------------------------------------------------------------

• oddjobd provides home directory creation when LDAP users log in the first time.

```
systemctl enable oddjobd --now
```

--------------------------------------------------------------------------------
## 8. Test LDAP Integration
--------------------------------------------------------------------------------

### 8.1 Test User Resolution
• Checks if the client can retrieve user details from LDAP.

```
getent passwd it1
```
Expected output:
```
it1:x:3003:2004:IT One:/home/it1:/bin/bash
```

### 8.2 Test Group Resolution
• Checks if the client can retrieve group details from LDAP.

```
getent group it
```
Expected output:
```
it:x:2004:it1,it2
```

### 8.3 Test Direct LDAP Connectivity
• Uses the ldapsearch tool to confirm direct communication with the LDAP server.

```
ldapsearch -x -H ldap://centos.armour.local -b "dc=armour,dc=local" "(uid=it1)"
```

--------------------------------------------------------------------------------
## 9. Troubleshooting Commands
--------------------------------------------------------------------------------

### 9.1 Clear SSSD Cache
• Removes cached data if LDAP entries are not updating or enumerating correctly.

```
systemctl stop sssd
```

```
rm -rf /var/lib/sss/db/*
```

```
rm -rf /var/lib/sss/mc/*
```

```
systemctl start sssd
```

### 9.2 Check SSSD Configuration
• Verifies the correctness of the SSSD configuration file.

```
sssctl config-check
```

### 9.3 Debug LDAP Authentication
• Tests whether a specific user can authenticate directly to the LDAP server.

```
ldapwhoami -x -D "uid=it1,ou=it,dc=armour,dc=local" -W -H ldap://centos.armour.local
```

---
