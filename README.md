# Student Registration System - CI/CD Pipeline (Flask + MongoDB on EC2)

A Flask + MongoDB student registration app, deployed via a fully automated
GitHub Actions pipeline: test -> build -> push to ECR -> deploy to EC2 -> health
check -> email notification.

## Architecture

```
push to main
   |
   v
[Test: pytest w/ MongoDB service] -> fails? pipeline stops here
   |
   v
[Build Docker image, tag = git commit SHA]
   |
   v
[Push image to Amazon ECR]
   |
   v
[Deploy to EC2: SSH in, docker pull, stop/rm old container, run new one]
   |
   v
[Verify: poll /health endpoint]
   |
   v
[Email: success or failure notification with build details]
```

## Prerequisites (set up manually before the pipeline runs)

### 1. AWS resources
- **ECR repository** to hold built images.
- **EC2 instance** (Ubuntu 22.04 recommended):
  - Docker installed and running
  - AWS CLI installed
  - Security group: inbound port `5000` open (app), port `22` restricted to
    your IP only (not `0.0.0.0/0`).
  - IAM permissions: attach an IAM instance role with
    `AmazonEC2ContainerRegistryReadOnly` (or scoped equivalent) so the instance
    can authenticate to ECR.

### 2. MongoDB
Use MongoDB Atlas (or self-hosted) and get a connection URI. The app connects
via `MONGO_URI` at runtime - it is never baked into the image.

### 3. SMTP for email notifications
Any SMTP provider works (Gmail with an App Password, SendGrid, Mailgun, etc).

## Required GitHub Secrets

Configure under **Repo Settings -> Secrets and variables -> Actions**:

| Secret | Purpose |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user with ECR push/pull permissions |
| `AWS_SECRET_ACCESS_KEY` | Paired with above |
| `AWS_REGION` | e.g. `us-east-1` |
| `ECR_REPOSITORY` | ECR repo name (e.g. `flask-practice`) |
| `EC2_HOST` | Public IP or DNS of the EC2 instance |
| `EC2_USER` | SSH user (e.g. `ubuntu`) |
| `EC2_SSH_KEY` | Private key (.pem contents) for SSH access |
| `MONGO_URI` | MongoDB connection string, injected into the container at deploy time |
| `APP_SECRET_KEY` | Flask secret key, injected into the container at deploy time |
| `SMTP_SERVER` | e.g. `smtp.gmail.com` |
| `SMTP_PORT` | e.g. `465` |
| `SMTP_USERNAME` | SMTP account username |
| `SMTP_PASSWORD` | SMTP account password / app password |
| `NOTIFY_EMAIL_TO` | Where success/failure emails get sent |

Never commit any of the above to the repo. `.env` is git-ignored; `.env.example`
shows the required shape for local development only.

## How the deploy step connects to EC2

**SSH-based.** The pipeline uses `appleboy/ssh-action` to SSH into the EC2
instance using a stored private key (`EC2_SSH_KEY` secret), then runs Docker
commands directly on the host: login to ECR, pull the new image, stop/remove
the old container, and start the new one with `--restart unless-stopped` so
a reboot doesn't kill the app.

SSH was chosen over SSM because it requires no extra IAM/SSM agent
configuration for a small assignment-scale deployment, and keeps the deploy
step transparent and easy to debug from the Actions log.

## How to reproduce a deployment manually (if the pipeline were unavailable)

```bash
# On your machine, authenticate Docker to ECR
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com

# Build and tag
docker build -t <account-id>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag> .
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>

# SSH into EC2
ssh -i your-key.pem ubuntu@<ec2-host>

# On the EC2 instance
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker pull <account-id>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>
docker stop flask-app || true
docker rm flask-app || true
docker run -d --name flask-app --restart unless-stopped -p 5000:5000 \
  -e MONGO_URI="<uri>" -e SECRET_KEY="<key>" \
  <account-id>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>

# Verify
curl http://<ec2-host>:5000/health
```

## Local development

```bash
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env       # fill in real MONGO_URI and SECRET_KEY
python app.py
```

Visit `http://localhost:5000`. Health check: `http://localhost:5000/health`.

## Local Docker test

```bash
docker build -t flask-practice .
docker run -p 5000:5000 --env-file .env flask-practice
curl http://localhost:5000/health
```

## Running tests

```bash
pytest -v
```

Requires a local MongoDB instance reachable at `mongodb://localhost:27017`
(the CI pipeline spins one up automatically as a service container).

## Features

- List, add, update, delete student records
- `/health` endpoint verifying live MongoDB connectivity (deploy-verification gate)
- Fully automated CI/CD: test gate -> Docker build (SHA-tagged) -> ECR push ->
  EC2 deploy -> health verification -> email notification (success/failure)

## License

MIT
