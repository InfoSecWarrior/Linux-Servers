# **Prepare CentOS 9 System for OpenLDAP Server**

This guide covers the initial system preparation and OpenLDAP installation on CentOS 9 for the domain `armour.local`.

---

## **1. Set System Hostname**

Configure the system hostname to match your LDAP server's fully qualified domain name:

```bash
hostnamectl set-hostname armour.local
```

This command sets the permanent hostname that will be used for:
- SSL/TLS certificate generation
- LDAP database naming conventions
- Client connection configurations
- Internal directory naming structure (`dc=armour,dc=local`)

### **Verify Hostname**
```bash
hostnamectl status
```

---

## **2. Configure Name Resolution**

You have two options for name resolution: using the hosts file (simple) or DNS (scalable).

### **Option A: Local Hosts File (Simple Method)**

Edit the hosts file to ensure local hostname resolution:

```bash
vim /etc/hosts
```

Add the following entry:
```
127.0.0.1   armour.local armour
```

This configuration ensures:
- Local resolution of the LDAP server hostname
- Proper functionality without external DNS dependency
- Quick setup for testing environments

### **Option B: DNS Configuration (Recommended for Production)**

#### **Method 1: Using Existing DNS Server**

If you have an existing DNS server in your network:

```bash
vim /etc/resolv.conf
```

Add your DNS server:
```
nameserver 192.168.1.10  # Replace with your DNS server IP
search armour.local
```

Then add the LDAP server record on your DNS server:
- **A Record**: `armour.local` → `192.168.1.100` (LDAP server IP)
- **PTR Record**: `192.168.1.100` → `armour.local` (reverse lookup)

#### **Method 2: Setup Local DNS Server**

Install and configure BIND as a local DNS server:

```bash
dnf install -y bind bind-utils
```

**Configure BIND:**

```bash
vim /etc/named.conf
```

Add to options section:
```
options {
    listen-on port 53 { 127.0.0.1; 192.168.1.100; };  # Add server IP
    allow-query { localhost; 192.168.1.0/24; };       # Allow network queries
    forwarders { 8.8.8.8; 8.8.4.4; };                # External DNS
};
```

**Create Zone File:**

```bash
vim /etc/named.conf.local
```

Add zone configuration:
```
zone "armour.local" {
    type master;
    file "/var/named/armour.local.zone";
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/var/named/1.168.192.rev";
};
```

**Create Forward Zone File:**

```bash
vim /var/named/armour.local.zone
```

```
$TTL 86400
@   IN  SOA armour.local. admin.armour.local. (
        2024010101  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400       ; Minimum TTL
)
    IN  NS  armour.local.
    IN  A   192.168.1.100

armour  IN  A   192.168.1.100
ldap    IN  CNAME   armour
```

**Create Reverse Zone File:**

```bash
vim /var/named/1.168.192.rev
```

```
$TTL 86400
@   IN  SOA armour.local. admin.armour.local. (
        2024010101  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400       ; Minimum TTL
)
    IN  NS  armour.local.

100 IN  PTR armour.local.
```

**Start DNS Service:**

```bash
systemctl enable named --now
```

```bash
firewall-cmd --add-service=dns --permanent
```

```bash
firewall-cmd --reload
```

**Update System to Use Local DNS:**

```bash
vim /etc/resolv.conf
```

```
nameserver 127.0.0.1
search armour.local
```

### **Verify Name Resolution**

Test forward lookup:
```bash
nslookup armour.local
```

```bash
dig armour.local
```

Test reverse lookup:
```bash
nslookup 192.168.1.100
```

```bash
dig -x 192.168.1.100
```

Test connectivity:
```bash
ping -c 2 armour.local
```

---

## **3. Install OpenLDAP Packages**

Install the required OpenLDAP server and client packages:

```bash
dnf install -y openldap-servers openldap-clients
```

**Package Details:**
- **openldap-servers** - Provides the LDAP server daemon (slapd) and server utilities
- **openldap-clients** - Provides client tools (ldapsearch, ldapadd, ldapmodify, etc.)

### **Verify Installation**
```bash
rpm -qa | grep openldap
```

---

## **4. Enable and Start LDAP Service**

Configure the slapd service to start automatically and launch it immediately:

```bash
systemctl enable slapd --now
```

This command:
- Enables automatic startup at system boot
- Starts the service immediately
- Creates necessary systemd links

### **Alternative Commands**
If you prefer separate commands:

```bash
systemctl enable slapd
```

```bash
systemctl start slapd
```

---

## **5. Verify Service Status**

Check that the LDAP server is running properly:

```bash
systemctl status slapd
```

Expected output should show:
- Active: active (running)
- Loaded: loaded
- Main PID information

### **Troubleshooting Failed Service**
If the service fails to start, examine the logs:

```bash
journalctl -xeu slapd
```

Common issues include:
- SELinux restrictions
- Configuration syntax errors
- Port conflicts

---

## **6. Configure Firewall Rules**

Open the necessary ports for LDAP communication:

### **Add LDAP Services**
```bash
firewall-cmd --add-service=ldap --permanent
```

```bash
firewall-cmd --add-service=ldaps --permanent
```

### **Reload Firewall**
```bash
firewall-cmd --reload
```

**Port Details:**
- **ldap** service opens TCP port 389 (standard LDAP)
- **ldaps** service opens TCP port 636 (LDAP over SSL/TLS)
- **--permanent** ensures rules persist after reboot

### **Verify Firewall Rules**
```bash
firewall-cmd --list-services
```

---

## **7. Verify LDAP Port Bindings**

Confirm that slapd is listening on the correct ports:

```bash
ss -tulpn | grep slapd
```

**Expected Output:**
```
tcp   LISTEN 0      128          0.0.0.0:389      0.0.0.0:*    users:(("slapd",pid=xxxx,fd=x))
tcp   LISTEN 0      128             [::]:389         [::]:*    users:(("slapd",pid=xxxx,fd=x))
```

### **Alternative Check**
```bash
netstat -tulpn | grep :389
```

---

## **8. Test Basic LDAP Connectivity**

Perform a basic anonymous search to verify the server is responding:

```bash
ldapsearch -x -H ldap://localhost -b "" -s base
```

This should return basic server information without errors.

### **Test Using Hostname**
```bash
ldapsearch -x -H ldap://armour.local -b "" -s base
```

---

## **System Preparation Summary**

| **Task** | **Purpose** | **Verification** |
|----------|-------------|------------------|
| Set hostname to `armour.local` | Establishes server identity for LDAP operations | `hostnamectl status` |
| Configure name resolution | Enables hostname lookup via hosts file or DNS | `nslookup armour.local` |
| Install OpenLDAP packages | Provides server daemon and management tools | `rpm -qa | grep openldap` |
| Enable and start slapd | Activates the LDAP server service | `systemctl status slapd` |
| Configure firewall | Allows network access to LDAP services | `firewall-cmd --list-services` |
| Verify port bindings | Confirms LDAP is accepting connections | `ss -tulpn | grep slapd` |

---

## **DNS vs Hosts File Comparison**

| **Aspect** | **Hosts File** | **DNS Server** |
|------------|----------------|----------------|
| **Setup Complexity** | Simple - edit one file | Complex - requires DNS service |
| **Scalability** | Limited - manual updates | Highly scalable - centralized |
| **Maintenance** | Edit each client manually | Update DNS server only |
| **Best For** | Small labs, testing | Production environments |
| **Dynamic Updates** | Not supported | Supported with DDNS |
| **Redundancy** | None | Multiple DNS servers possible |

---

## **Next Steps**

After completing these preparation steps, the system is ready for:
- LDAP administrator password configuration
- Directory schema setup
- Organizational structure creation
- User and group management
- SSL/TLS configuration

---

## **Important Notes**

- Ensure SELinux is properly configured or set to permissive mode during initial setup
- The domain `armour.local` will translate to `dc=armour,dc=local` in LDAP entries
- All LDAP tools will now reference this server as `ldap://armour.local`
- For production environments, always use DNS instead of hosts file entries
- Consider implementing SSL/TLS immediately after basic configuration

---
