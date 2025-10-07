🎮 Game Database Management System



Welcome to the Game Database Management System project! This is a user-friendly application built with Streamlit and MySQL, allowing users to manage, update, and explore a database of video games. The project is designed to help game enthusiasts and developers keep track of game details, including titles, genres, developers, platforms, and pricing.

📋 Table of Contents

Features

# Game Database Management System

This repository contains a Streamlit-based web app for managing a video game inventory backed by a MySQL database. The README below explains the project layout, how to run it locally, and multiple ways to deploy the app to AWS (EC2, Elastic Beanstalk with Docker, or App Runner).

Repository: https://github.com/Amitkumarracha/AWS--Game-Database-Management-System

## Table of contents
- Project overview
- Repository structure
- Requirements & prerequisites
- Local setup and run
- Database setup (local and AWS RDS)
- AWS deployment options (EC2, Elastic Beanstalk with Docker, App Runner)
- GitHub repo & CI/CD notes
- Security & next steps

## Project overview

A small, user-friendly Streamlit application to add, update, view, and delete games from a MySQL-backed inventory. The UI lives in `app.py` and pages are under the `pages/` folder. Database helper functions are in `db_utils.py` and `database.py`.

## Repository structure

Top-level files and important folders:

- `app.py` — Streamlit application entrypoint and navigation.
- `requirements.txt` — Python dependencies (minimal, editable).
- `DATABASE SQL.sql` — SQL script to create the `project` database and tables.
- `database.py` — helper to create DB connection and initialize tables (uses Faker for sample data).
- `db_utils.py` — CRUD functions used by pages.
- `config.toml` — Streamlit theme settings.
- `pages/` — Streamlit pages:
  - `homepage.py`, `add_game.py`, `update_delete_game.py`, `game_inventory.py`, `About.py`.
- `images/` — static images used by the UI.

## Requirements & prerequisites

- Python 3.8+ (project uses 3.12 in the local venv)
- MySQL server (local) or an AWS RDS MySQL instance
- Git
- AWS account (for deployment)

We included a minimal `requirements.txt`. Install dependencies into a virtualenv:

PowerShell example:

```powershell
Set-Location -Path 'D:\AWS- Game-Database-Management-System'
python -m venv .venv
& .venv\Scripts\python.exe -m pip install --upgrade pip
& .venv\Scripts\python.exe -m pip install -r requirements.txt
```

## Local setup and run

1. Clone the repo:

```powershell
git clone https://github.com/Amitkumarracha/AWS--Game-Database-Management-System.git
Set-Location -Path 'AWS--Game-Database-Management-System'
```

2. Create venv and install dependencies (see previous section).

3. Ensure MySQL is running and create the database schema (or let `database.py` create tables):

```powershell
mysql -u root -p < 'D:\AWS- Game-Database-Management-System\DATABASE SQL.sql'
```

Or run `database.initialize_database()` by executing `database.py` once; note this file calls `initialize_database()` on import.

4. Run the app:

```powershell
& '.venv\Scripts\python.exe' -m streamlit run app.py
```

Open http://localhost:8501 in your browser.

## Database configuration / environment variables

Currently DB credentials are hardcoded in `database.py` and `db_utils.py`. Before deploying to production you should move credentials to environment variables. Recommended variable names:

- `DB_HOST`
- `DB_USER`
- `DB_PASSWORD`
- `DB_NAME`

Example snippet (Python) to read env vars:

```python
import os

DB_CONFIG = {
    'user': os.environ.get('DB_USER', 'root'),
    'password': os.environ.get('DB_PASSWORD', ''),
    'host': os.environ.get('DB_HOST', 'localhost'),
    'database': os.environ.get('DB_NAME', 'project')
}
```

On Windows (PowerShell) you can set them like this for the current process:

```powershell
$env:DB_HOST = 'your-rds-endpoint'
$env:DB_USER = 'admin'
$env:DB_PASSWORD = 'strong-password'
$env:DB_NAME = 'project'
```

On Linux (systemd or Docker) use `export VAR=...` or pass via the container/task configuration.

## Database on AWS — RDS recommended

1. Create an AWS RDS MySQL instance (choose appropriate instance size, set a secure master user and password).
2. Make sure the RDS security group allows inbound traffic from your application server (EC2, EB, or App Runner). Prefer limiting by IP or security group rather than 0.0.0.0/0.
3. Run `DATABASE SQL.sql` against the RDS endpoint to create tables, or allow `database.py` to create them automatically (but only after you update credentials to use environment variables).

## Deploying to AWS

Below are three practical options. Choose the one that fits your experience and budget.

### Option A — EC2 (fast, flexible)

1. Launch an EC2 instance (Amazon Linux 2 / Ubuntu). Configure Security Group to allow SSH (22), HTTP (80) and/or your Streamlit port (8501) if not using a reverse proxy.
2. SSH into the instance, install system packages, Python, and MySQL client:

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git nginx
```

3. Clone the repo, create venv, install requirements, set env variables, and run the app in background or via systemd.

Example `systemd` unit to run Streamlit behind Nginx (file `/etc/systemd/system/streamlit.service`):

```ini
[Unit]
Description=Streamlit app
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/AWS--Game-Database-Management-System
Environment=DB_HOST=your-rds-endpoint
Environment=DB_USER=admin
Environment=DB_PASSWORD=SuperSecret
Environment=DB_NAME=project
ExecStart=/home/ubuntu/AWS--Game-Database-Management-System/.venv/bin/python -m streamlit run app.py --server.port 8501 --server.headless true
Restart=always

[Install]
WantedBy=multi-user.target
```

4. Use Nginx as a reverse proxy from port 80 to 8501:

```nginx
server {
    listen 80;
    server_name your.server.domain; # or public IP

    location / {
        proxy_pass http://127.0.0.1:8501;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

5. Start and enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now streamlit.service
```

### Option B — Elastic Beanstalk with Docker (recommended for simple app containers)

1. Create a `Dockerfile` in project root (example below) and an `Dockerrun.aws.json`/EB config as needed.

Example Dockerfile (simple):

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . /app
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 8501
CMD ["streamlit", "run", "app.py", "--server.port", "8501", "--server.headless", "true"]
```

2. Initialize and deploy with Elastic Beanstalk (EB CLI):

```bash
eb init -p docker my-game-db-app --region us-east-1
eb create my-game-db-app-env
eb deploy
```

3. Configure environment variables in the EB console (Configuration -> Software -> Environment properties) to set DB credentials and other secrets.

### Option C — AWS App Runner or ECS/Fargate (recommended for managed containers)

- Build a Docker image and push to ECR.
- Create an App Runner service or ECS task referencing that image.
- Configure environment variables for DB credentials and set up a VPC connection to RDS if RDS is private.

## GitHub repo and pushing

If you want to push the local repo to your GitHub repository URL:

```powershell
git remote add origin https://github.com/Amitkumarracha/AWS--Game-Database-Management-System.git
git branch -M main
git push -u origin main
```

## CI/CD (GitHub Actions) — outline

- You can create a GitHub Actions workflow to build and push a Docker image to ECR on push to `main`, then trigger an EB or ECS deployment. A minimal workflow will:
  - Checkout code
  - Build Docker image
  - Log in to ECR
  - Push image
  - Optionally call AWS CLI to update service or trigger EB deploy

If you'd like, I can add a sample `.github/workflows/deploy-to-eb.yml` or `deploy-to-ecr.yml` for you.

## Security notes

- Never commit DB credentials to the repository. Use environment variables or AWS Secrets Manager.
- Add `.env` and other local configuration files to `.gitignore`.

## Next steps I can implement for you

- Add `.gitignore` and a `.env.example` file.
- Switch `db_utils.py` / `database.py` to read DB credentials from environment variables.
- Add a `Dockerfile` and an example `systemd` service file.
- Add a GitHub Actions workflow to build and push a container to ECR and deploy to Elastic Beanstalk or ECS.

Tell me which of the above you'd like me to add or implement next and I will create the files and make the code changes.
