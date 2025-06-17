# Implementing LDAPS on CentOS 9
---

This document guides how to upgrade your LDAP server to LDAPS.

-------------------------------------------------------------------------------
## 1. Create SSL Directory Structure
-------------------------------------------------------------------------------
• Creates a dedicated location to store your LDAP server’s certificates.

```
mkdir -p /etc/openldap/certs
```

```
cd /etc/openldap/certs
```

-------------------------------------------------------------------------------
## 2. Generate Private Key and Certificate
-------------------------------------------------------------------------------
• Uses OpenSSL to generate a self-signed certificate (valid for 365 days) and a private key. Adjust the -subj fields (C=, ST=, etc.) as appropriate.

```
openssl req -new -x509 -nodes -out ldapserver.crt -keyout ldapserver.key -days 365 \
  -subj "/C=IN/ST=MP/L=City/O=Armour/CN=centos.armour.local"
```

-------------------------------------------------------------------------------
## 3. Set Proper Ownership and Permissions
-------------------------------------------------------------------------------
• Ensures that only the “ldap” user can read the private key (600), while the certificate can be world-readable (644).

```
chown ldap:ldap /etc/openldap/certs/*
```
```
chmod 600 /etc/openldap/certs/ldapserver.key
```
```
chmod 644 /etc/openldap/certs/ldapserver.crt
```

-------------------------------------------------------------------------------
## 4. Copy Certificate to Client Trust Store
-------------------------------------------------------------------------------
• Updates the local CA trust so that local tools (like ldapsearch) can trust the server’s certificate.

```
cp /etc/openldap/certs/ldapserver.crt /etc/pki/ca-trust/source/anchors/
```
```
update-ca-trust
```

-------------------------------------------------------------------------------
## 5. Configure OpenLDAP for TLS/SSL
-------------------------------------------------------------------------------

### 5.1 Create TLS Configuration File
• Defines the paths to the CA certificate file, server certificate file, and private key file for slapd.

```
vim /tmp/tls-config.ldif
```
```ldif
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/openldap/certs/ldapserver.crt
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/ldapserver.crt
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/ldapserver.key
```

### 5.2 Apply TLS Configuration
• Applies the LDIF file to enable SSL/TLS in the LDAP server configuration.

```
ldapmodify -Y EXTERNAL -H ldapi:/// -f /tmp/tls-config.ldif
```

-------------------------------------------------------------------------------
## 6. Configure LDAP to Listen on LDAPS Port
-------------------------------------------------------------------------------

### 6.1 Edit slapd Environment Variables
• Instructs slapd to listen on the local socket (ldapi:///), port 389 (ldap://), and port 636 (ldaps://).

```
vim /etc/sysconfig/slapd
```
```bash
SLAPD_URLS="ldapi:/// ldap:/// ldaps:///"
```
• ldapi:/// → Local socket (for admin commands)  
• ldap:/// → Standard LDAP port 389  
• ldaps:/// → Encrypted LDAP on port 636  

### 6.2 Restart slapd
• Restarts the LDAP daemon to apply the new configuration.

```
systemctl restart slapd
```
```
systemctl status slapd
```

### 6.3 Verify LDAPS Port
• Checks whether slapd is listening on TCP port 636.

```
ss -nltp | grep 636
```

### 6.4 Open LDAPS in Firewall
• Allows incoming LDAPS connections on port 636 through the system firewall.

```
firewall-cmd --add-service=ldaps --permanent
```
```
firewall-cmd --reload
```

-------------------------------------------------------------------------------
## 7. Test Local LDAPS Connection
-------------------------------------------------------------------------------
• Performs a secure LDAP search on port 636 to confirm encrypted communication.

```
ldapsearch -x -H ldaps://centos.armour.local -b "dc=armour,dc=local" -D "cn=admin,dc=armour,dc=local" -W
```

-------------------------------------------------------------------------------
## 8. Debug Connection if Needed
-------------------------------------------------------------------------------
• Uses OpenSSL to inspect certificate details and the TLS handshake with the server.

```
openssl s_client -connect centos.armour.local:636 -showcerts
```

-------------------------------------------------------------------------------
## 9. Update Server-Side Services for LDAPS
-------------------------------------------------------------------------------
If you are running SSSD on the same LDAP server, update it to prefer ldaps://.

### 9.1 Update SSSD Configuration
• Modifies /etc/sssd/sssd.conf to point SSSD to the secure LDAP port 636, referencing the server’s certificate.

```
vim /etc/sssd/sssd.conf
```
```ini
[domain/armour.local]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldaps://centos.armour.local:636
ldap_search_base = dc=armour,dc=local
ldap_schema = rfc2307
ldap_tls_reqcert = allow
ldap_tls_cacert = /etc/openldap/certs/ldapserver.crt
ldap_id_use_start_tls = false
# Remove this line if present: ldap_auth_disable_tls_never_use_in_production = true
cache_credentials = true
enumerate = true
ldap_default_bind_dn = cn=admin,dc=armour,dc=local
ldap_default_authtok = @rmour123
```

### 9.2 Restart SSSD
• Reloads SSSD with the updated settings, then expires cached data to ensure immediate effect.

```
systemctl restart sssd
```
```
sssctl cache-expire -E
```

-------------------------------------------------------------------------------
## 10. Update Client Configuration for LDAPS
-------------------------------------------------------------------------------

### 10.1 Copy LDAP Certificate to Client
• Transfers the server’s TLS certificate to the client so the client can verify the server’s identity.

On the server:
```
scp /etc/openldap/certs/ldapserver.crt root@c1.armour.local:/tmp/
```

On the client:
```
cp /tmp/ldapserver.crt /etc/pki/ca-trust/source/anchors/
```
```
update-ca-trust
```

### 10.2 Update Client SSSD Configuration
• Edits the client’s /etc/sssd/sssd.conf to use ldaps:// and references the CA certificate.

```
vim /etc/sssd/sssd.conf
```
```ini
[domain/armour.local]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldaps://centos.armour.local:636
ldap_search_base = dc=armour,dc=local
ldap_schema = rfc2307
ldap_tls_reqcert = allow
ldap_tls_cacert = /etc/pki/ca-trust/source/anchors/ldapserver.crt
ldap_id_use_start_tls = false
# Remove: ldap_auth_disable_tls_never_use_in_production = true
cache_credentials = true
enumerate = true
ldap_default_bind_dn = cn=admin,dc=armour,dc=local
ldap_default_authtok = @rmour123
```

### 10.3 Restart SSSD on Client
• Applies the updated configuration and clears old cache.

```
systemctl restart sssd
```
```
sssctl cache-expire -E
```

### 10.4 Test Authentication
• Ensures user resolution now occurs over an encrypted LDAP connection.

```
getent passwd it1
```

-------------------------------------------------------------------------------
## 11. Verify Complete LDAPS Setup
-------------------------------------------------------------------------------

### 11.1 Check Server-Side LDAPS Port
• Confirms that port 636 is listening and open.

```
ss -nltp | grep 636
```

```
netstat -an | grep :636
```

### 11.2 Monitor LDAP Logs
• Observes live logs to confirm or troubleshoot LDAPS connections.

```
journalctl -u slapd -f
```

-------------------------------------------------------------------------------
## 12. Disable Plain LDAP (Optional)
-------------------------------------------------------------------------------
• Restricts slapd to only LDAPS connections after confirming success.

```
vim /etc/sysconfig/slapd
```
```bash
SLAPD_URLS="ldapi:/// ldaps:///"
```

```
systemctl restart slapd
```

-------------------------------------------------------------------------------
## 13. Troubleshooting Common Issues
-------------------------------------------------------------------------------

### 13.1 Certificate Issues
• Validate certificate details and debug LDAPS connection.

```bash
openssl x509 -in /etc/openldap/certs/ldapserver.crt -text -noout
```
```
ldapsearch -x -H ldaps://centos.armour.local -b "" -s base -d 1
```

### 13.2 SSSD Issues
• Increase debug logging in /etc/sssd/sssd.conf (debug_level = 9) and check logs:

```bash
journalctl -u sssd -f
```
```
tail -f /var/log/sssd/*.log
```

### 13.3 Clear All Caches
• Removes outdated entries if there are issues with user/group enumeration.

```bash
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

---
