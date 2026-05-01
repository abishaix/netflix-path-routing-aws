# Netflix — AWS ALB Path-Based Routing

A Netflix-inspired multi-service web app deployed on AWS using Application Load Balancer path-based routing. Each section of the site runs on a **dedicated private EC2 server** — traffic is routed by the ALB based on the URL path.

Built as a hands-on project for the **DevOps Bootcamp** to demonstrate real-world ALB architecture.

---

## 🏗️ Architecture

```
User → walmart.com (or ALB DNS)
           │
           ▼
    Internet Gateway
           │
           ▼
  Application Load Balancer
  (internet-facing, spans 2 AZs)
           │
    ┌──────┼──────┬──────────┬────────────┐
    ▼      ▼      ▼          ▼            ▼
  path:/ path:/trending path:/series path:/movies path:/new
    │      │      │          │            │
    ▼      ▼      ▼          ▼            ▼
 TG-home TG-trending TG-series TG-movies TG-new
    │      │      │          │            │
    ▼      ▼      ▼          ▼            ▼
 EC2-1  EC2-2  EC2-3      EC2-4        EC2-5
(Nginx) (Nginx)(Nginx)   (Nginx)      (Nginx)
private private private   private      private
subnet  subnet  subnet    subnet       subnet
```

---

## 📋 AWS Resources

| Resource | Details |
|---|---|
| VPC | Custom — 10.0.0.0/16 |
| Public Subnets | 2 — one per AZ (for ALB nodes) |
| Private Subnets | 2 — one per AZ (for app servers) |
| Internet Gateway | Attached to VPC |
| NAT Gateway | Regional — private servers can install packages |
| Security Group (public) | Port 80 + 22 inbound |
| Security Group (private) | Port 80 from public SG only |
| EC2 Instances | 5x t3.micro — Nginx on each |
| Target Groups | 5 — one per service |
| ALB | Internet-facing, HTTP:80 |
| ALB Listener Rules | 4 path rules + 1 default |

---

## 🎬 Services & Routing Rules

| Path | Target Group | Server | Content |
|---|---|---|---|
| `/` (default) | TG-home | EC2-home | Homepage — all sections |
| `/trending` | TG-trending | EC2-trending | Trending Now |
| `/series` | TG-series | EC2-series | TV Series |
| `/movies` | TG-movies | EC2-movies | Movies |
| `/new` | TG-new | EC2-new | New Releases |

---

## 🚀 Deployment

### Step 1 — Clone this repo
```bash
git clone https://github.com/abishaix/netflix-path-routing-aws.git
cd netflix-path-routing-aws
```

### Step 2 — Build AWS infrastructure
1. Create VPC with 2 public + 2 private subnets across 2 AZs
2. Attach Internet Gateway, create NAT Gateway
3. Configure route tables (public → IGW, private → NAT)
4. Create Security Groups
5. Launch 5 EC2 instances (t3.micro) in private subnets

### Step 3 — Deploy content to each server

SSH into each server via bastion host, then:

```bash
sudo su -
yum install nginx -y
systemctl start nginx
systemctl enable nginx
```

**Home server:**
```bash
cp index.html /usr/share/nginx/html/index.html
systemctl restart nginx
```

**Trending server:**
```bash
mkdir /usr/share/nginx/html/trending
cp trending/index.html /usr/share/nginx/html/trending/index.html
systemctl restart nginx
```

Repeat for `/series`, `/movies`, `/new`.

### Step 4 — Update ALB DNS in index.html

After creating the ALB, copy its DNS name and update the nav links in `index.html`:

```html
<!-- Replace ALB_DNS_NAME with your actual ALB DNS -->
<a href="http://ALB_DNS_NAME/trending">Trending</a>
<a href="http://ALB_DNS_NAME/series">TV Series</a>
<a href="http://ALB_DNS_NAME/movies">Movies</a>
<a href="http://ALB_DNS_NAME/new">New Releases</a>
```

### Step 5 — Create Target Groups

For each service, create a Target Group:
- Type: Instances
- Protocol: HTTP, Port: 80
- Health check path:
  - Home: `/`
  - Trending: `/trending`
  - Series: `/series`
  - Movies: `/movies`
  - New: `/new`

Register the correct EC2 in each TG.

### Step 6 — Create Application Load Balancer

1. Type: Application Load Balancer
2. Scheme: Internet-facing
3. Select both public subnets (different AZs)
4. Add listener rules:

| Condition | Action |
|---|---|
| path = `/trending` | Forward to TG-trending |
| path = `/series` | Forward to TG-series |
| path = `/movies` | Forward to TG-movies |
| path = `/new` | Forward to TG-new |
| Default | Forward to TG-home |

---

## 🧹 Cleanup Order

To avoid charges, delete in this order:

1. Load Balancer
2. Target Groups
3. EC2 Instances (all 5 + bastion)
4. NAT Gateway
5. Release Elastic IPs
6. Security Groups
7. Route Tables
8. Subnets
9. Internet Gateway (detach first)
10. VPC

---

## 💡 Key Concepts Demonstrated

- **Path-based routing** — one ALB, multiple services, each on its own server
- **Private subnet deployment** — app servers have no public IP, only accessible via ALB
- **Target Groups** — isolate which servers the LB can route to
- **Health checks** — LB monitors each server independently
- **Fault isolation** — if one server goes down, only that section is affected
- **NAT Gateway** — private servers can reach the internet for installs without being exposed

---

## 📁 Repo Structure

```
netflix-path-routing-aws/
├── index.html          ← Homepage (TG-home)
├── trending/
│   └── index.html      ← Trending (TG-trending)
├── series/
│   └── index.html      ← TV Series (TG-series)
├── movies/
│   └── index.html      ← Movies (TG-movies)
└── new/
    └── index.html      ← New Releases (TG-new)
```

---

## 🔗 Connect

- GitHub: [abishaix](https://github.com/abishaix)
- LinkedIn: [linkedin.com/in/abimxai](https://linkedin.com/in/abimxai)

---

*Built during DevOps Bootcamp — Week 3*
