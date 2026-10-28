# FORGOTTEN — AWS EC2 Deployment

Runs the Discord **bot** and the **wallet-verification website** on a single small
EC2 instance, sharing one SQLite database, with free automatic HTTPS. Designed to
fit the AWS free tier (≈ $0 for the first 12 months).

```
                 Internet
          https://verify.forgottennft.com
                    │  (443)
            ┌───────▼────────┐  EC2 (t3.micro, Ubuntu 22.04)
            │     Caddy       │  auto Let's Encrypt TLS + HSTS + CSP
            └───────┬────────┘
                    │ 127.0.0.1:5080 (localhost only)
          ┌─────────▼──────────┐      ┌────────────────────┐
          │ Forgotten.WalletWeb│      │   Forgotten.Bot    │  → Discord (outbound)
          └─────────┬──────────┘      └─────────┬──────────┘  → Solana DAS (outbound)
                    └────────── /var/lib/forgotten/forgotten.db ──────────┘
```

Only **80/443** (web) and **22** (SSH, your IP only) are open. The bot needs no
inbound ports.

---

## 0. Security principles (do not skip)

- **Never share AWS/root credentials or access keys in chat or with third parties.**
  Use a dedicated least-privilege **IAM user** + **MFA on the root account**.
- Secrets live only in `/etc/forgotten/forgotten.env` on the box (`chmod 600`,
  owned by the `forgotten` user) — never in git or Terraform.
- SSH is key-only and restricted to your IP. Services run as a non-root user with
  systemd hardening. HTTPS is enforced (HSTS).
- Rotate any key that has ever been exposed (e.g. the Helius key) before putting the
  fresh value in `forgotten.env`.

## 1. Prerequisites (local, one time)

- The repo checked out, with the code **committed** (don't deploy uncommitted work).
- Tools: AWS CLI v2, Terraform ≥ 1.6, .NET 10 SDK, Node 18+, Git Bash (Windows).
- In AWS: create an **IAM user** with programmatic access and a policy covering EC2 +
  VPC + EIP (e.g. attach `AmazonEC2FullAccess` to start; tighten later). Enable MFA on
  root. Then authenticate the CLI locally:
  ```bash
  aws configure            # paste the IAM user's key/secret + region (local only; never in chat)
  aws sts get-caller-identity   # confirm it's the IAM user, not root
  ```
- Create (or import) an **EC2 key pair** in your chosen region and note its name and
  the `.pem` file path.

## 2. Provision the instance (Terraform)

```bash
cd deploy/terraform
cp terraform.tfvars.example terraform.tfvars   # edit: region, key_name, ssh_allowed_cidr (your IP/32), domain, acme_email
terraform init
terraform plan        # review what will be created
terraform apply       # creates the EC2 + Elastic IP + security group
```

Note the `public_ip` output.

## 3. DNS at GoDaddy

Add an **A record** for the subdomain → the Elastic IP:

| Type | Name   | Value (Elastic IP) | TTL |
|------|--------|--------------------|-----|
| A    | verify | `<public_ip>`      | 600 |

(Keep the apex `forgottennft.com` / `www` on your marketing site.) Wait for it to
resolve: `nslookup verify.forgottennft.com`.

## 4. Put secrets on the box

```bash
ssh -i <your-key.pem> ubuntu@<public_ip>
sudo cp /dev/stdin /etc/forgotten/forgotten.env   # paste contents based on deploy/forgotten.env.example, then Ctrl-D
sudo chown forgotten:forgotten /etc/forgotten/forgotten.env
sudo chmod 600 /etc/forgotten/forgotten.env
```

Fill in a **fresh** Discord token + Helius DAS endpoint, and set
`WalletVerification__PublicBaseUrl=https://verify.forgottennft.com`.

Optional (all in the same env file; see `deploy/forgotten.env.example`):

| Variable | Effect |
|---|---|
| `Discord__DevGuildId` | Guild-scoped command registration (instant) + required for the verify-log announcer |
| `Discord__FlexRoleId` | Restrict `/nft showcase` + flex to a holders role |
| `Discord__VerifyLogChannelId` | Post a "Wallet Verified" embed to this channel on each verification |
| `WalletVerification__TipLinkClientId` | Enables the TipLink wallet button (clientId is request-only from TipLink) |

## 5. Publish & deploy the apps

From the **repo root** (local):

```bash
deploy/publish-and-deploy.sh <public_ip> <your-key.pem>
# ARM (t4g) instead? prefix with RID=linux-arm64
```

This builds the WalletWeb client bundle, publishes both apps self-contained, uploads
them, and starts the `forgotten-bot` + `forgotten-walletweb` systemd services. The
database schema is created automatically on first start (EF Core migrations).

## 6. Verify

```bash
curl -sS https://verify.forgottennft.com/health      # {"status":"ok",...}
ssh -i <your-key.pem> ubuntu@<public_ip> 'journalctl -u forgotten-bot -n 30 --no-pager'
```

In Discord the bot should be online; `/wallet connect` now returns a
`https://verify.forgottennft.com/verify/...` link that works for any member.

## 7. Updating later

Commit changes, then re-run step 5 (`publish-and-deploy.sh`). It replaces the
binaries and restarts the services; pending migrations apply on startup.

## 8. Backups & housekeeping

**Database backups** run in two layers:

- **On-instance (daily):** `deploy/forgotten-backup.sh` → `/usr/local/bin/forgotten-backup.sh`,
  invoked by `/etc/cron.d/forgotten-backup` at 03:17 UTC. It takes a WAL-safe `sqlite3 .backup`,
  gzips it to `/var/lib/forgotten/backups/`, and keeps the 14 most recent.
- **Off-instance (S3):** `deploy/terraform/backups.tf` provisions a private, versioned,
  encrypted S3 bucket + a least-privilege **IAM instance profile** (no keys on the box). When
  `BACKUP_S3_BUCKET` is set in `/etc/forgotten/backup.env`, the same script also uploads each
  backup to `s3://<bucket>/db/` (30-day lifecycle). This survives loss of the instance/volume.

Restore: `aws s3 cp s3://<bucket>/db/<file>.gz .`, `gunzip`, stop walletweb, drop it in as
`/var/lib/forgotten/forgotten.db`, restart.

- Logs: `journalctl -u forgotten-bot` / `-u forgotten-walletweb` / `-u caddy`.
- Costs: free-tier covers `t3.micro` for 12 months. After that, `t4g.micro`
  (ARM, publish `linux-arm64`) is the cheapest. Set a small AWS **Budget alert**.

## 9. Tear down

```bash
cd deploy/terraform && terraform destroy
```
Then remove the GoDaddy A record. (Back up the DB first if you want to keep data.)
