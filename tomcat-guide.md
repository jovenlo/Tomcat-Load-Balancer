# Tomcat 10.1.42 — Installation, Configuration & Web Page Setup

Complete guide to install Apache Tomcat 10.1.42 on Amazon Linux 2023,
configure it, deploy a web page, and verify it is running.

---

## Prerequisites

- Amazon Linux 2023 EC2 instance
- Java 21 installed
- SSH access to the server

---

## Step 1 — Update the System

```bash
sudo yum update -y
```

---

## Step 2 — Install Java 21

```bash
sudo yum install -y java-21-amazon-corretto
java -version
```

Expected output:
```
openjdk version "21.x.x" ...
```

---

## Step 3 — Download Tomcat 10.1.42

```bash
cd /opt
sudo wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.42/bin/apache-tomcat-10.1.42.tar.gz
```

---

## Step 4 — Extract and Rename

```bash
sudo tar -xzf apache-tomcat-10.1.42.tar.gz
sudo mv apache-tomcat-10.1.42 tomcat
sudo rm -f apache-tomcat-10.1.42.tar.gz
```

Tomcat is now installed at `/opt/tomcat`

---

## Step 5 — Set Permissions

```bash
sudo chmod -R 755 /opt/tomcat
sudo chmod +x /opt/tomcat/bin/*.sh
```

---

## Step 6 — Start Tomcat

```bash
sudo /opt/tomcat/bin/startup.sh
```

Expected output:
```
Tomcat started.
```

---

## Step 7 — Verify Tomcat is Running

```bash
# Check the process
ps aux | grep tomcat

# Check port 8080 is listening
ss -tlnp | grep 8080

# Test HTTP response
curl -I http://localhost:8080
```

Expected:
```
HTTP/1.1 200 OK
```

---

## Step 8 — Create a Custom Web Page

```bash
# Create a new web application folder
sudo mkdir -p /opt/tomcat/webapps/myapp

# Create a simple HTML page
sudo tee /opt/tomcat/webapps/myapp/index.html > /dev/null <<EOF
<!DOCTYPE html>
<html>
  <head>
    <title>My Tomcat App</title>
  </head>
  <body>
    <h1>Hello from Tomcat 10.1.42!</h1>
    <p>Server is running successfully on port 8080.</p>
  </body>
</html>
EOF
```

---

## Step 9 — Access the Web Page

Open your browser and go to:

```
http://<your-EC2-public-IP>:8080/myapp/
```

You should see:
```
Hello from Tomcat 10.1.42!
Server is running successfully on port 8080.
```

---

## Step 10 — Set Up Tomcat as a systemd Service (Auto-start)

```bash
sudo tee /etc/systemd/system/tomcat.service > /dev/null <<EOF
[Unit]
Description=Apache Tomcat 10
After=network.target

[Service]
Type=forking
Environment=JAVA_HOME=/usr/lib/jvm/java-21-amazon-corretto
Environment=CATALINA_HOME=/opt/tomcat
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl enable tomcat
sudo systemctl status tomcat
```

---

## Useful Commands

```bash
# Start Tomcat
sudo systemctl start tomcat

# Stop Tomcat
sudo systemctl stop tomcat

# Restart Tomcat
sudo systemctl restart tomcat

# Check status
sudo systemctl status tomcat

# View Tomcat logs
sudo tail -f /opt/tomcat/logs/catalina.out

# Shutdown manually
sudo /opt/tomcat/bin/shutdown.sh

# Startup manually
sudo /opt/tomcat/bin/startup.sh
```

---

## Directory Structure

```
/opt/tomcat/
├── bin/          # startup.sh, shutdown.sh scripts
├── conf/         # server.xml, web.xml configuration files
├── logs/         # catalina.out log file
├── webapps/      # Deploy your apps here
│   ├── ROOT/     # Default root app
│   └── myapp/    # Your custom app
└── work/         # Tomcat working directory
```

---

## Default Ports

| Service | Port |
|---|---|
| Tomcat HTTP | 8080 |
| Tomcat HTTPS | 8443 |
| Tomcat Shutdown | 8005 |
| Tomcat AJP | 8009 |

---

## Change Tomcat Port (Optional)

To change the default port from 8080 to another port, edit `server.xml`:

```bash
sudo nano /opt/tomcat/conf/server.xml
```

Find this line:
```xml
<Connector port="8080" protocol="HTTP/1.1"
```

Change `8080` to your desired port, then restart:
```bash
sudo systemctl restart tomcat
```

---

## Troubleshooting

| Problem | Solution |
|---|---|
| Port 8080 not accessible | Check EC2 Security Group — add inbound rule port 8080 |
| Tomcat not starting | Check logs: `tail -f /opt/tomcat/logs/catalina.out` |
| Java not found | Run `java -version` — reinstall if missing |
| Permission denied | Run `sudo chmod -R 755 /opt/tomcat` |
