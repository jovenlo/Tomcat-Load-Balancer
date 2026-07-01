# SSL Setup — AWS ALB + Tomcat + JBoss

Secure your AWS Load Balancer with SSL using a self-signed certificate.
No domain name required. Backend servers stay on plain HTTP — zero changes needed.

---

## Architecture

```
User
  │
  │  HTTPS (port 443) 🔒 Encrypted
  ▼
Application Load Balancer  ← SSL Certificate lives here
  │
  │  HTTP (plain, internal VPC only)
  ├──▶ LinuxServer1 — Tomcat 10.1.42 (port 8080)
  └──▶ LinuxServer2 — JBoss WildFly 32 (port 9090)
```

---

## How SSL Termination Works

| Layer | Protocol | Port | Encrypted |
|---|---|---|---|
| User → ALB | HTTPS | 443 | ✅ Yes |
| ALB → Tomcat | HTTP | 8080 | ❌ Internal only |
| ALB → JBoss | HTTP | 9090 | ❌ Internal only |

SSL is handled entirely at the ALB. Backend servers need no changes.

---

## Prerequisites

- Two EC2 instances running:
  - LinuxServer1 → Tomcat 10.1.42 on port 8080
  - LinuxServer2 → JBoss WildFly 32 on port 9090
- An existing ALB with Target Groups
- OpenSSL installed on your local machine

---

## Step 1 — Create Self-Signed Certificate

Run these commands on your local machine or any EC2 instance:

```bash
# Generate private key
openssl genrsa -out my-private-key.pem 2048

# Generate self-signed certificate (valid 365 days)
openssl req -new -x509 \
  -key my-private-key.pem \
  -out my-certificate.pem \
  -days 365 \
  -subj "/C=US/ST=State/L=City/O=MyOrg/CN=my-alb"
```

This creates:
- `my-private-key.pem` — keep this safe, never share
- `my-certificate.pem` — your SSL certificate

---

## Step 2 — Upload Certificate to AWS IAM

1. Go to **AWS Console → IAM**
2. On the left menu click **Server Certificates** (under Security Tools)
3. Click **Import**
4. Fill in:
   - **Certificate name** → `my-self-signed-cert`
   - **Certificate body** → paste contents of `my-certificate.pem`
   - **Private key** → paste contents of `my-private-key.pem`
5. Click **Import**

> To view certificate contents run:
> ```bash
> cat my-certificate.pem
> cat my-private-key.pem
> ```

---

## Step 3 — Update ALB Security Group

1. Go to **AWS Console → EC2 → Security Groups**
2. Click your **ALB Security Group**
3. Click **Edit Inbound Rules**
4. Click **Add Rule** and set:

| Type | Protocol | Port | Source |
|---|---|---|---|
| HTTPS | TCP | 443 | 0.0.0.0/0 |

5. Click **Save Rules**

---

## Step 4 — Add HTTPS Listener to ALB

1. Go to **AWS Console → EC2 → Load Balancers**
2. Click your **ALB**
3. Click **Listeners and rules** tab
4. Click **Add Listener**
5. Set:
   - Protocol → **HTTPS**
   - Port → **443**
   - Default action → **Forward**
   - Target group → select your existing Target Group
6. Scroll down to **Secure listener settings**:
   - Security policy → leave as default
   - Certificate source → **From IAM**
   - Certificate → select **my-self-signed-cert**
7. Click **Add**

---

## Step 5 — Redirect HTTP to HTTPS

1. Go to **Listeners and rules** tab
2. Click on the **HTTP:80** listener
3. Click **Edit**
4. Change Default action to **Redirect to URL**
5. Set:
   - Protocol → **HTTPS**
   - Port → **443**
   - Status code → **301 - Permanently moved**
6. Click **Save changes**

Now anyone visiting `http://` is automatically sent to `https://`.

---

## Step 6 — Test the Setup

Open your browser and go to:
```
https://<your-ALB-DNS-Name>
```

> You will see a browser security warning — this is normal for self-signed certificates.
> - Chrome → click **Advanced** → **Proceed to site**
> - Firefox → click **Advanced** → **Accept the Risk and Continue**

The connection is still fully encrypted — warning is only because
the certificate was not issued by a trusted authority.

Test from command line:
```bash
# Test HTTPS
curl -k https://<ALB-DNS-Name>

# Test HTTP redirect to HTTPS
curl -I http://<ALB-DNS-Name>
```

Expected redirect response:
```
HTTP/1.1 301 Moved Permanently
Location: https://...
```

---

## Full Traffic Flow

```
User → http://ALB-DNS
            │
     301 Redirect (port 80)
            │
User → https://ALB-DNS
            │
    ALB decrypts SSL 🔒
            │
      ┌─────┴─────┐
      ▼           ▼
 Tomcat:8080  JBoss:9090
 (HTTP plain) (HTTP plain)
```

---

## Troubleshooting

| Problem | Solution |
|---|---|
| HTTPS not reachable | Add port 443 to ALB Security Group inbound rules |
| Browser security warning | Normal for self-signed — click Advanced and proceed |
| curl SSL error | Use `curl -k` flag to skip cert verification |
| HTTP not redirecting | Check HTTP:80 listener action is set to Redirect |
| Certificate not showing in IAM | Re-upload — make sure you pasted full contents of the .pem files |
| Backend unhealthy | Health checks still use HTTP — no changes needed on EC2 |

---

## Notes

- Self-signed certificates are free but show a browser warning
- For no browser warning, get a real domain and use **AWS ACM** (free)
- Certificate expires after 365 days — recreate and re-upload when expired
- Never share or commit your `my-private-key.pem` to GitHub

---

## Files in This Project

```
.
├── tomcat-guide.md                # Tomcat installation guide
├── apache-httpd-guide.md          # Apache HTTPD guide
├── tomcat-jboss-integration.md    # Integration guide
├── install-jboss.sh               # JBoss install script
├── ssl-setup-guide.md             # Detailed SSL steps
└── README.md                      # This file
```

---

## License

MIT
