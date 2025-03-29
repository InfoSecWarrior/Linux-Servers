# Nginx Server Installation and Configuration on CentOS

## **Step 1: Update System Packages**
Before installing Nginx, update your system:

```bash
yum update -y
```

---

## **Step 2: Install Nginx**

Enable the EPEL repository (if not already enabled):

```bash
yum install epel-release -y
```

Install Nginx:

```bash
yum install nginx -y
```

Start and enable Nginx at boot:

```bash
systemctl start nginx
systemctl enable nginx
```

Check the status:

```bash
systemctl status nginx
```

---

## **Step 3: Configure Firewall**

Allow HTTP and HTTPS traffic:

```bash
firewall-cmd --permanent --add-service=http
```
```bash
firewall-cmd --permanent --add-service=https
```
```bash
firewall-cmd --reload
```

---

## **Step 4: Verify Installation**

Open a web browser and go to:

```
http://your-server-ip
```

If Nginx is running, you should see the default Nginx welcome page.

---

## **Step 5: Configure Nginx Server Blocks (Virtual Hosts)**

### **1. Create a Directory for Your Website**

```bash
mkdir -p /var/www/html/index.html
```
```bash
chown -R apache:apache /var/www/html/index.html
```
```bash
chmod -R 755 /var/www/html/
```

### **2. Create a Sample HTML Page**

```bash
echo "<h1>Welcome to example.com</h1>" | sudo tee /var/www/example.com/html/index.html
```

### **3. Create a Server Block Configuration File**

```bash
vim /etc/nginx/conf.d/example.com.conf
```

Add the following configuration:

```bash
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/example.com/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Save and exit.

### **4. Restart Nginx**

```bash
systemctl restart ngnix
```

---

## **Step 6: Managing Nginx**

- Restart Nginx:
  ```bash
  systemctl restart nginx
  ```
- Stop Nginx:
  ```bash
  systemctl stop nginx
  ```
- Reload Nginx without dropping connections:
  ```bash
  systemctl reload nginx
  ```

Your Nginx server is now set up on CentOS! 🚀

