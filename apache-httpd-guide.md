# Apache HTTPD — Installation, Configuration & Tomcat Migration

Complete guide to install Apache HTTPD on Amazon Linux 2023,
configure it to display a web page, and set it up as a reverse proxy
in front of Tomcat (migration/integration).

---

## Prerequisites

- Amazon Linux 2023 EC2 instance
- SSH access to the server
- Tomcat running on port 8080 (see `tomcat-guide.md`)

---

## Part 1 — Install & Configure Apache HTTPD

---

### Step 1 — Update the System

```bash
sudo yum update -y
```

---

### Step 2 — Install Apache HTTPD

```bash
sudo yum install -y httpd
httpd -v
```

Expected output:
```
Server version: Apache/2.4.x (Amazon Linux)
```

---

### Step 3 — Start and Enable Apache

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
```

---

### Step 4 — Verify Apache is Running

```bash
# Check port 80 is listening
ss -tlnp | grep 80

# Test HTTP response
curl -I http://localhost
```

Expected:
```
HTTP/1.1 200 OK
```

---

### Step 5 — Create a Custom Web Page

```bash
# Create your HTML page in the Apache web root
sudo tee /var/www/html/index.html > /dev/null <<EOF
<!DOCTYPE html>
<html>
  <head>
    <title>My Apache Web Page</title>
  </head>
  <body>
    <h1>Hello from Apache HTTPD!</h1>
    <p>Apache is running successfully on port 80.</p>
  </body>
</html>
EOF
```

---

### Step 6 — Access the Web Page

Open your browser and go to:

```
http://<your-EC2-public-IP>
```

You should see:
```
Hello from Apache HTTPD!
Apache is running successfully on port 80.
```

---

## Part 2 — Migrate Apache to Work with Tomcat (Reverse Proxy)

This configures Apache HTTPD to sit in front of Tomcat.
All traffic comes in on port 80 (Apache) and is forwarded to Tomcat on port 8080.

```
User → Apache (port 80) → Tomcat (port 8080)
```

---

### Step 7 — Enable Required Apache Modules

```bash
# Verify proxy modules are available
httpd -M | grep proxy
```

Expected output should include:
```
proxy_module
proxy_http_module
```

These are enabled by default on Amazon Linux 2023. If missing:
```bash
sudo yum install -y mod_proxy
```

---

### Step 8 — Configure Reverse Proxy

```bash
# Add reverse proxy config to Apache
sudo tee /etc/httpd/conf.d/tomcat-proxy.conf > /dev/null <<EOF
<VirtualHost *:80>

    # Forward all traffic to Tomcat
    ProxyPreserveHost On
    ProxyPass / http://localhost:8080/
    ProxyPassReverse / http://localhost:8080/

    # Logs
    ErrorLog /var/log/httpd/tomcat-error.log
    CustomLog /var/log/httpd/tomcat-access.log combined

</VirtualHost>
EOF
```

---

### Step 9 — Fix SELinux Permission

Amazon Linux 2023 has SELinux enabled. Without this, Apache cannot connect to Tomcat:

```bash
sudo setsebool -P httpd_can_network_connect 1
```

Verify it is set:
```bash
getsebool httpd_can_network_connect
```

Expected:
```
httpd_can_network_connect --> on
```

---

### Step 10 — Test Apache Config & Restart

```bash
# Test for syntax errors
sudo apachectl configtest

# Restart Apache
sudo systemctl restart httpd
sudo systemctl status httpd
```

---

### Step 11 — Verify the Reverse Proxy is Working

```bash
# This should now return Tomcat's response through Apache on port 80
curl -I http://localhost
```

Expected:
```
HTTP/1.1 200 OK
```

Open your browser and go to:

```
http://<your-EC2-public-IP>
```

You should now see the **Tomcat welcome page** served through Apache on port 80 — no need to type `:8080` anymore.

---

### Step 12 — Deploy Your App Through Apache → Tomcat

Deploy your WAR file to Tomcat as usual:

```bash
sudo cp /tmp/myapp.war /opt/tomcat/webapps/
```

Then access it via Apache:

```
http://<your-EC2-public-IP>/myapp/
```

Apache forwards the request to Tomcat automatically.

---

## Useful Commands

```bash
# Start Apache
sudo systemctl start httpd

# Stop Apache
sudo systemctl stop httpd

# Restart Apache
sudo systemctl restart httpd

# Check status
sudo systemctl status httpd

# Test config for errors
sudo apachectl configtest

# View Apache access logs
sudo tail -f /var/log/httpd/access_log

# View Apache error logs
sudo tail -f /var/log/httpd/error_log

# View reverse proxy logs
sudo tail -f /var/log/httpd/tomcat-access.log
sudo tail -f /var/log/httpd/tomcat-error.log
```

---

## Directory Structure

```
/etc/httpd/
├── conf/
│   └── httpd.conf          # Main Apache config file
├── conf.d/
│   └── tomcat-proxy.conf   # Your reverse proxy config
└── modules/                # Apache modules

/var/www/html/              # Apache web root (static files)
/var/log/httpd/             # Apache log files
```

---

## Default Ports

| Service | Port |
|---|---|
| Apache HTTP | 80 |
| Apache HTTPS | 443 |
| Tomcat (behind Apache) | 8080 (internal only) |

---

## How the Migration Works

```
BEFORE (Direct Tomcat):
User → EC2:8080 → Tomcat → Response

AFTER (Apache + Tomcat):
User → EC2:80 → Apache → localhost:8080 → Tomcat → Response
```

Benefits of putting Apache in front of Tomcat:
- Users access port 80 (standard HTTP) instead of 8080
- Apache handles static files faster
- Easy to add SSL/HTTPS on Apache later
- Better logging and access control

---

## Troubleshooting

| Problem | Solution |
|---|---|
| Port 80 not accessible | Check EC2 Security Group — add inbound rule port 80 |
| 502 Bad Gateway | Tomcat is not running — start it: `sudo systemctl start tomcat` |
| SELinux blocking connection | Run: `sudo setsebool -P httpd_can_network_connect 1` |
| Apache config error | Run: `sudo apachectl configtest` to find the error |
| Page not updating | Clear browser cache or try `curl http://localhost` |
| Apache not starting | Check logs: `sudo tail -f /var/log/httpd/error_log` |
