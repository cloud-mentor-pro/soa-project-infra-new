# soa — AWS Infrastructure (CloudFormation)

Hạ tầng AWS cho backend SOA project — quản lý hoàn toàn bằng CloudFormation, không có bước
tạo resource thủ công qua console.

Kiến trúc: VPC 3-tier (public / private-app / private-db) → ECS Fargate chạy sau Application
Load Balancer (HTTPS, TLS terminate tại ALB) → RDS PostgreSQL đặt cô lập ở private-db subnet,
không có đường ra internet. Domain và Certificate (ACM, DNS-validated qua Route53) tự động gắn
vào ALB. Một Bastion EC2 phục vụ vận hành và migrate DB, truy cập qua SSM Session Manager —
không SSH, không public IP.

Mật khẩu DB và secret key của app đi qua Secrets Manager và SSM Parameter Store, không hardcode
trong template hay task definition ở bất kỳ đâu. Hạ tầng tách thành 7 stack theo layer
(networking, security, domain, database, config, compute, bastion), mỗi stack một trách nhiệm,
deploy theo đúng thứ tự phụ thuộc.

Project được tổ chức để làm việc hiệu quả với **Claude Code** — có đầy đủ CLAUDE.md, rules,
skills, commands, agents trong thư mục `.claude/`.

---

## Architecture

```
Internet
    │
    ▼
[Route53]  api-dev.{your-domain}  (A record → ALB alias, tạo tự động trong stack compute)
    │
    ▼
[ALB]  internet-facing  ·  Public Subnet 1a + 1c
  :80  → redirect 301 → :443
  :443 → forward → Target Group  (ACM cert từ stack domain, TLS 1.3)
    │
    ├──▶ [ECS Fargate task]  Private App Subnet 1a
    └──▶ [ECS Fargate task]  Private App Subnet 1c
              │
              ├──▶ [NAT Gateway]  →  Internet (ECR pull, package updates)
              │
              └──▶ [RDS PostgreSQL]  Private DB Subnet 1a/1c (isolated, no NAT route)
                        │
                        └──▶ credentials in Secrets Manager (ManageMasterUserPassword)

[Bastion EC2]  Private App Subnet 1a  (no public IP, no KeyPair)
  → SSM Session Manager only
  → connect-db.sh reads db-url (SSM) + credentials (Secrets Manager) automatically
```

---

## Stack Topology

7 stack, deploy theo thứ tự — `domain` là ngoại lệ duy nhất không phụ thuộc gì cả nên có thể
deploy song song với `networking`:

```
1. {env}-soa-networking   →  templates/networking/soa-networking.yaml
2. {env}-soa-security     →  templates/security/soa-security.yaml
3. {env}-soa-domain       →  templates/domain/soa-domain.yaml       (độc lập, deploy sớm)
4. {env}-soa-database     →  templates/database/soa-database.yaml
5. {env}-soa-config       →  templates/config/soa-config.yaml
6. {env}-soa-compute      →  templates/compute/soa-compute.yaml     (cần domain + database + config)
7. {env}-soa-bastion      →  templates/bastion/soa-bastion.yaml
```

Hướng dẫn deploy từng bước, điền parameters, migrate DB, teardown và troubleshooting — xem
[DEPLOY.md](DEPLOY.md).

---

## Project Structure

```
soa-project-infra-new/
├── templates/
│   ├── networking/soa-networking.yaml   ← VPC 3-tier, NAT GW, Route Tables, S3 Endpoint
│   ├── security/soa-security.yaml       ← Security Groups (ALB/ECS/DB/Bastion)
│   ├── domain/soa-domain.yaml           ← ACM Certificate api.{domain}, DNS-validated
│   ├── database/soa-database.yaml       ← RDS PostgreSQL, Secrets Manager managed password
│   ├── config/soa-config.yaml           ← SSM Parameters, tự động từ database stack
│   ├── compute/soa-compute.yaml         ← ECS Fargate, ALB (HTTPS), ECR, Route53, Auto Scaling
│   └── bastion/soa-bastion.yaml         ← EC2 SSM-only, helper scripts
├── parameters/
│   ├── dev-{layer}.json        ← 7 file
│   └── prod-{layer}.json       ← 7 file
├── .claude/
│   ├── CLAUDE.md               ← Project context cho Claude Code
│   ├── CLAUDE.local.md         ← Personal overrides (gitignored)
│   ├── outputs-catalog.md      ← Danh sách CloudFormation exports
│   ├── rules/                  ← cfn-standards, security-guardrails, data-protection, env-separation
│   ├── skills/                 ← new-stack, cross-stack-ref, iam-audit, cost-guard
│   ├── commands/                ← /project:deploy, validate, diff, rollback, drift, security-scan
│   └── agents/                  ← security-auditor, cost-analyzer
├── DEPLOY.md                    ← Hướng dẫn deploy từ đầu đến cuối
└── README.md
```

---

## Thiết kế Stack

Giá trị dưới đây là default cho **dev** (`parameters/dev-*.json`) — prod dùng instance lớn hơn
và bật HA, xem chi tiết trong từng file `parameters/prod-*.json`.

### 1. Networking
- VPC `10.0.0.0/16`, 6 subnet trải 2 AZ
- 2 public subnet — đặt ALB, NAT Gateway
- 2 private-app subnet — đặt ECS, Bastion
- 2 private-db subnet — đặt RDS, cô lập hoàn toàn, không có route ra internet
- NAT Gateway: 1 cái (dev) / 2 cái, mỗi AZ 1 (prod, HA)
- S3 Gateway Endpoint — subnet private truy cập S3 không qua NAT

### 2. Security
4 Security Group:
- **ALB** — mở port 80/443 ra internet
- **ECS** — chỉ nhận traffic từ ALB SG
- **Bastion** — không có ingress rule nào (SSM là agent-initiated, không cần mở port vào)
- **DB** — chỉ nhận port 5432 từ ECS SG và Bastion SG

### 3. Domain
- 1 ACM Certificate cho `api-dev.{domain}`
- `ValidationMethod: DNS` — CloudFormation tự tạo CNAME validate qua Route53, không thao tác tay

### 4. Database
- Engine: **PostgreSQL 18.3**
- Instance: `db.t3.micro` (dev), single instance (không Multi-AZ)
- Storage: 20GB `gp3`, có mã hoá
- Master password: **không tự đặt** — Secrets Manager tự sinh và quản lý (`ManageMasterUserPassword`)
- Backup: `BackupRetentionPeriod: 1` ngày (dev)
- Monitoring: Performance Insights + export log `postgresql` sang CloudWatch
- `DeletionPolicy: Delete` — xoá stack là mất DB thật, không giữ snapshot (chủ đích, xem DEPLOY.md)

### 5. Config
3 SSM Parameter (`/soa/dev/*`):
- `secret-key` — Flask `SECRET_KEY`, nhập tay 1 lần lúc deploy
- `db-url` — hostname RDS, tự động lấy từ stack Database
- `db-secret-arn` — ARN secret chứa username/password, tự động lấy từ stack Database

### 6. Compute
- **ECR** — repo riêng, giữ tối đa 10 image gần nhất
- **ECS Fargate** cluster (Container Insights bật sẵn), task mặc định `256 CPU / 512 MB`,
  `DesiredCount: 0` (chờ CI/CD push image lần đầu)
- Credentials container (DB user/password, secret key) inject lúc runtime qua ECS `Secrets`
  (Secrets Manager + SSM) — không hardcode trong task definition
- **ALB** internet-facing: `:80` redirect sang `:443`, `:443` forward tới Target Group
  (TLS 1.3, cert từ stack Domain), health check tại `/users/ping`
- **Auto Scaling** theo CPU 60% / Memory 70%, `MinCapacity: 0` → `MaxCapacity: 5` (dev)
- **Route53** A-record alias trỏ thẳng vào ALB
- 2 IAM Role riêng (Task Execution, Task Role) — least-privilege, chỉ đọc đúng secret/parameter
  cần thiết

### 7. Bastion
- EC2 `t3.micro`, Amazon Linux 2023, đặt ở private-app subnet
- **Không public IP, không KeyPair** — chỉ truy cập qua SSM Session Manager
- Cài sẵn: PostgreSQL 18 client, Git, Node.js LTS + `pnpm`, Docker + Docker Compose + `buildx`,
  vài tool cơ bản (`htop`, `tree`, `vim`, `jq`...)
- 2 helper script tự sinh lúc boot:
  - `connect-db.sh` — tự lấy credentials từ Secrets Manager rồi `psql` vào RDS, không cần nhập tay
  - `get-soa-params.sh` — liệt kê nhanh toàn bộ SSM parameter của env

---

## Tham khảo

- [DEPLOY.md](DEPLOY.md) — Hướng dẫn deploy chi tiết từng bước
- [.claude/CLAUDE.md](.claude/CLAUDE.md) — Quyết định kiến trúc, naming convention
- [.claude/outputs-catalog.md](.claude/outputs-catalog.md) — CloudFormation export names
- [AWS CloudFormation Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html)
