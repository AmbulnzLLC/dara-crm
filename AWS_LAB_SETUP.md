# AWS Lab Account Deployment

This documents the test deployment of Twenty CRM in the AWS lab account
(`510526718390`, profile `lab`, region `us-west-2`), set up on 2026-06-12.

## What's running

A single EC2 instance runs the entire stack via Docker Compose — there is no
RDS, ElastiCache, or S3. Everything lives on the instance's EBS root volume.

| Resource | Value |
|----------|-------|
| EC2 instance | `i-03e4d58957e534812` — t3.medium, Amazon Linux 2023, 30GB gp3 |
| Public IP | `34.214.48.164` (not an Elastic IP — changes on stop/start) |
| App URL | http://34.214.48.164:3000 |
| VPC / subnet | `vpc-e2885a9a` (devops_vpc_1, default VPC), `subnet-59af8320` (us-west-2a) |
| Security group | `twenty-crm-test` (`sg-097d050da331c060d`) |
| IAM instance profile | `aws-ec2-role-for-ssm` (SSM Session Manager access) |
| Tags | `Name=twenty-crm-test`, `Owner=acambone`, `Purpose=twenty-crm-testing` |

The compose stack (in `/opt/twenty` on the instance) runs four containers:

- **server** — `twentycrm/twenty:latest`, the API + frontend on port 3000
- **worker** — same image, background job processor
- **db** — Postgres 16, data in the `twenty_db-data` Docker volume
- **redis** — cache and job queue, no persistence

File uploads use `STORAGE_TYPE=local` (the `twenty_server-local-data` volume),
not S3.

## Network access

- Port **3000** is open only to a single allowed IP (originally
  `52.34.96.225/32`). No other inbound ports are open — no SSH.
- Instance management is via **SSM Session Manager**:

  ```bash
  aws ssm start-session --profile lab --target i-03e4d58957e534812
  ```

- If your public IP changes, add it to the security group:

  ```bash
  aws ec2 authorize-security-group-ingress --profile lab \
    --group-id sg-097d050da331c060d \
    --protocol tcp --port 3000 --cidr <your-ip>/32
  ```

## Secrets

`ENCRYPTION_KEY` and the Postgres password were generated randomly at first
boot and stored in `/opt/twenty/.env` (mode 600) on the instance. They exist
nowhere else — if you rebuild the instance, new ones are generated and any
old database volume becomes unreadable (encrypted fields won't decrypt).

## How it was built

1. Created the security group in the default VPC and allowed port 3000 from a
   single IP:

   ```bash
   aws ec2 create-security-group --profile lab \
     --group-name twenty-crm-test \
     --description "Twenty CRM test instance" \
     --vpc-id vpc-e2885a9a

   aws ec2 authorize-security-group-ingress --profile lab \
     --group-id <sg-id> --protocol tcp --port 3000 --cidr <your-ip>/32
   ```

2. Launched the instance with a user-data script that installs Docker,
   generates secrets, writes the compose file, and starts the stack:

   ```bash
   aws ec2 run-instances --profile lab \
     --image-id ami-09e69ca1171857250 \
     --instance-type t3.medium \
     --subnet-id subnet-59af8320 \
     --security-group-ids <sg-id> \
     --iam-instance-profile Name=aws-ec2-role-for-ssm \
     --associate-public-ip-address \
     --user-data file://twenty-user-data.sh \
     --block-device-mappings 'DeviceName=/dev/xvda,Ebs={VolumeSize=30,VolumeType=gp3,DeleteOnTermination=true}' \
     --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=twenty-crm-test},{Key=Owner,Value=acambone},{Key=Purpose,Value=twenty-crm-testing}]'
   ```

   The AMI was the latest Amazon Linux 2023 at the time, resolved via:

   ```bash
   aws ssm get-parameter --profile lab \
     --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
     --query 'Parameter.Value' --output text
   ```

3. The user-data script (also saved at `/var/lib/cloud/instance/user-data.txt`
   on the instance) does the following:

   - `dnf install docker` + installs the compose CLI plugin
   - Generates `/opt/twenty/.env` with `SERVER_URL` (from instance metadata),
     a random `ENCRYPTION_KEY` (`openssl rand -base64 32`), and a random
     Postgres password
   - Writes a docker-compose.yml based on
     [`packages/twenty-docker/docker-compose.yml`](packages/twenty-docker/docker-compose.yml)
     from this repo (trimmed to the env vars actually used)
   - Runs `docker compose up -d`

   First boot takes ~4-5 minutes (package install + image pulls) before the
   app answers on `/healthz`.

## Day-2 operations

```bash
# Shell on the instance
aws ssm start-session --profile lab --target i-03e4d58957e534812

# Once on the instance — manage the stack
cd /opt/twenty
sudo docker compose ps          # status
sudo docker compose logs -f     # logs
sudo docker compose pull && sudo docker compose up -d   # upgrade to latest
sudo docker compose restart     # restart
```

Data persists across reboots and stop/start (Docker volumes on EBS).
Stop/start changes the public IP — update `SERVER_URL` in `/opt/twenty/.env`
and `docker compose up -d` again, plus the security group rule if needed.

## Teardown

```bash
aws ec2 terminate-instances --profile lab --instance-ids i-03e4d58957e534812
# after the instance is gone:
aws ec2 delete-security-group --profile lab --group-id sg-097d050da331c060d
```

The EBS volume (and all data) is deleted with the instance.

## Known limitations (fine for testing, fix before real use)

- Plain HTTP, no TLS — put an ALB with an ACM cert or nginx + Let's Encrypt
  in front before exposing it more broadly
- No backups — Postgres lives only on the instance volume
- Public IP is ephemeral — attach an Elastic IP if the URL needs to be stable
- Single instance — Postgres should move to RDS and uploads to S3
  (`STORAGE_TYPE=s3`) if this becomes a shared environment
