# **Linux DNS Server Setup (CentOS 9 & Debian 12)**

## **1. Introduction to DNS**

The **Domain Name System (DNS)** converts domain names into IP addresses, allowing users to access websites easily. Instead of remembering numbers, DNS lets us use names like **armour.local** for **192.168.31.60**. This guide covers setting up a DNS server on **CentOS 9** and **Debian 12**.

---

## **2. Installing DNS Server**

### **CentOS 9**

1. **Check if BIND is installed:**
   ```bash
   rpm -qa | grep bind
   ```
2. **Install BIND (DNS Server):**
   ```bash
   yum install bind bind-utils -y
   ```
3. **Verify Installation:**
   ```bash
   rpm -qi bind        # Display package details  
   rpm -ql bind        # List installed files  
   ```

### **Debian 12**

1. **Update package list:**
   ```bash
   apt update
   ```
2. **Install BIND:**
   ```bash
   apt install bind9 bind9-utils -y
   ```
3. **Check Service Status:**
   ```bash
   systemctl status bind9
   ```

---

## **3. Configuring DNS Server**

### **Edit the Main Configuration File**

#### **CentOS 9:**

```bash
nano /etc/named.conf
```

#### **Debian 12:**

```bash
nano /etc/bind/named.conf.options
```

Modify or add the following lines:

```bash
options {
    listen-on port 53 { 192.168.31.60; };
    allow-query { any; };
    recursion yes;
};
```

### **Create Forward Zone File**

#### **CentOS 9:**

```bash
nano /etc/named/zones/db.armour.local
```

#### **Debian 12:**

```bash
nano /etc/bind/db.armour.local
```

Add these entries:

```
$TTL 86400
@   IN  SOA armour.local. root.armour.local. (
        2024030601  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400 )     ; Minimum TTL

@       IN  NS  ns1.armour.local.
ns1     IN  A   192.168.31.60
www     IN  A   192.168.31.60
```

### **Create Reverse Zone File**

#### **CentOS 9:**

```bash
nano /etc/named/zones/db.31.168.192
```

#### **Debian 12:**

```bash
nano /etc/bind/db.31.168.192
```

Add these entries:

```
$TTL 86400
@   IN  SOA armour.local. root.armour.local. (
        2024030601  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400 )     ; Minimum TTL

@       IN  NS  ns1.armour.local.
60      IN  PTR ns1.armour.local.
```

---

## **4. Configuring Secondary DNS Server (Slave)**

### **On the Primary DNS Server (192.168.31.60)**

Edit the configuration file:

```bash
nano /etc/bind/named.conf.local
```

Add:

```
zone "armour.local" {
    type master;
    file "/etc/bind/db.armour.local";
    allow-transfer { 192.168.31.61; };
    also-notify { 192.168.31.61; };
};
```

Restart the service:

```bash
systemctl restart bind9
```

### **On the Secondary DNS Server (192.168.31.61)**

Install BIND:

```bash
apt install bind9 bind9-utils -y
```

Edit the configuration file:

```bash
nano /etc/bind/named.conf.local
```

Add:

```
zone "armour.local" {
    type slave;
    file "/var/cache/bind/db.armour.local";
    masters { 192.168.31.60; };
};
```

Restart the service:

```bash
systemctl restart bind9
```

---

## **5. Configure DNS Clients**

Edit the resolver configuration file on client machines:

```bash
nano /etc/resolv.conf
```

Add:

```
nameserver 192.168.31.60
domain armour.local
search armour.local
```

---

## **6. Restart and Enable Services**

#### **CentOS 9:**

```bash
systemctl restart named
systemctl enable named
systemctl status named
```

#### **Debian 12:**

```bash
systemctl restart bind9
systemctl enable bind9
systemctl status bind9
```

---

## **7. Testing the DNS Server**

On any machine, test the DNS server:

```bash
dig @192.168.31.60 www.armour.local
nslookup www.armour.local 192.168.31.60
```

If the results display the correct IP your DNS server is up and running! 

---

## **8. Troubleshooting DNS Issues**

### **Check Service Status**

```bash
systemctl status named   # CentOS 9
systemctl status bind9   # Debian 12
```

### **Check for Configuration Errors**

```bash
named-checkconf      # CentOS 9
named-checkzone armour.local /etc/bind/db.armour.local   # Debian 12
```

### **Verify DNS Query Resolution**

```bash
dig armour.local
nslookup armour.local
```

### **Check if Port 53 is Open**

```bash
netstat -tulnp | grep :53
ss -tulnp | grep :53
```

### **Check Logs for Errors**

#### **CentOS 9:**

```bash
tail -f /var/log/messages
```

#### **Debian 12:**

```bash
tail -f /var/log/syslog
```

