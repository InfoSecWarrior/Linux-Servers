# DNS Server Configuration on Debian

## Prerequisites
- A Debian-based server with root or sudo access.
- A static IP address configured.
---

## Step 1: Update your system:
Update your system

```bash
apt update
```
---

## Step 2: Install BIND9
Install `bind9`:

```bash
apt install bind9 -y
```

Start and enable the service:

```bash
systemctl start bind9
systemctl enable bind9
```

Check the status:

```bash
systemctl status bind9
```

---

## Step 3: Configure BIND9

### Edit the Main Configuration File
Open `/etc/bind/named.conf.options`:

```bash
vim /etc/bind/named.conf.options
```

Modify the `options` block:

```bash
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };
};
```

Save and exit.

---

## Step 4: Create a Zone File

Edit `/etc/bind/named.conf.local` to define your zone:

```bash
sudo nano /etc/bind/named.conf.local
```

## **Configure the Forward DNS Zone**

```bash
zone "example.com" {
    type master;
    file "/etc/bind/zones/db.example.com";
};
```


Create the zone directory and zone file:

```bash
mkdir -p /etc/bind/zones
vim /etc/bind/zones/db.example.com
```

Add the following content:

```bash
$TTL 604800
@   IN  SOA example.com. admin.example.com. (
        1 ; Serial
        604800 ; Refresh
        86400 ; Retry
        2419200 ; Expire
        604800 ) ; Negative Cache TTL
;
@   IN  NS  ns1.example.com.
ns1 IN  A   192.168.1.100
www IN  A   192.168.1.101

A Record (Address Record)
Maps a domain name to an IPv4 address.

example.com.   IN  A   192.168.1.100

AAAA Record (IPv6 Address Record)
example.com.   IN  AAAA  2001:db8::1

CNAME Record (Canonical Name Record)
www.example.com.   IN  CNAME   example.com.

MX Record (Mail Exchange Record)
example.com.   IN  MX  10 mail.example.com.

TXT Record (Text Record)
example.com.   IN  TXT  "v=spf1 mx -all"
```

Save and exit.

---

## **Configure the Reverse DNS Zone**
Edit the BIND configuration file to define the reverse zone:

```bash
vim /etc/bind/named.conf.local
```

Add the following lines:

```plaintext
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.1";
};
```

Save and exit.

---

## **Create the Reverse Zone File**

Create the reverse zone file:

```bash
vim /etc/bind/zones/db.192.168.1
```

Add the following content:

```plaintext
$TTL 604800
@   IN  SOA  ns1.example.com. admin.example.com. (
        2024032901 ; Serial
        7200       ; Refresh
        3600       ; Retry
        1209600    ; Expire
        86400 )    ; Minimum TTL
;
@   IN  NS  ns1.example.com.
1   IN  PTR server1.example.com.
2   IN  PTR server2.example.com.
```

Save and exit.

---


## Step 4: Check Configuration and Restart BIND

Verify the configuration:

```bash
named-checkconf
named-checkzone example.com /etc/bind/zones/db.example.com
```

Restart the BIND service:

```bash
systemctl restart bind9
```

---

## Step 5: Test the DNS Server

Use `dig` or `nslookup` to test:

```bash
dig @192.168.1.50(server IP) example.com
```

or

```bash
nslookup @192.168.1.50(server IP) example.com
```

If everything is correct, your DNS server is now successfully configured on Debian! 🚀

